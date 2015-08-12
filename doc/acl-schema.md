# Root用户不受访问控制的任何约束

# 表格的访问控制信息在表格的schema中
```
TableSchema {
String read_group;  // 此处存储group name
String write_group;
String admin_group; // disable, delete, drop, update
}
```

e.g.

```
linkbase {
read_group  reader;
write_group writer;
admin_group admin;
}
```

# 用户信息存在meta表中

**token -> user_name ->group**

每个用户都对应一个唯一的token，这个token会存在于RPC请求中，
token用于决定用户身份（此RPC请求是张三还是李四发起的），再将user映射到group中，
user所属的group将决定它的具体权限（如是否有权限写linkbase）。

**Key**: ~user_name

**Value**: 是一个pb结构

```
message {
string token;
repeated string group_name;
}
```

master在内存中维护`token->user`和`user->group`的映射，在读取meta表时初始化。

# 访问控制
表格ACL和用户信息持久化存储在meta表中。

1. master可以在内存中保存一份，每个RPC请求查内存数据就可以决定是否允许操作；
1. ts不能在每个RPC请求到达时查meta表，因为性能太差。

解决方案：

1. master在表格ACL和用户信息变更时通知ts
1. master在表格ACL和用户信息变更时写到nexus或zk上，ts对以上数据执行watch以更新（bigtable和hbase的方案）

方案1. master直接通知ts

信息的初始化：ts加入集群后master通知ts；

信息的更新：master在表格ACL和用户信息变更时通知ts

优点：简单直白，rpc+内存操作

不足：哪些信息该往哪些ts上推送的逻辑由master实现，加大master复杂度？

方案2. master写到nexus或zk

信息的初始化：用户信息在ts初始化时获取，表格信息在load时获取（只获取需要的表格信息）

信息的更新：watch-更新

1. acl目录下有一个叶子结点user，存储所有用户信息；如果用户数量较多，可以优化。
1. acl目录下再有一个tables目录结点，其下很多子结点，每个table对应一个叶子结点，值为该表格的schema.

优点：master只负责更新信息到nexus/zk，ts按需自取，主要逻辑由ts实现

不足：实现起来有点复杂？有很多和nexus/zk交互的问题

例如：当一个write的RPC到达ts，ts从rpc中取出token，影射到user，再映射到group，对比内存中的表格schema.

# 用户接口

设置表格的读group组：

./teracli acl table-read table-name read-group-name

write/admin类同

将user添加到group中：

./teracli acl addtogroup user-name group-name

从group中删除类同

创建用户：

./teracli acl adduser user password

删除用户类同
