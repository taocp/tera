# Root用户不受访问控制的任何约束

# 表格的访问控制信息在表格的schema中
```
TableSchema {
repeated String who_can_read;
repeated String who_can_write;
repeated String who_can_admin; // disable, delete, drop, update
}
```

# 用户信息存在meta表中
每个用户都对应一个唯一的token，这个token会存在于RPC请求中，token用于决定用户身份（此RPC请求是张三还是李四发起的）。

**Key**: ~token

**Value**: 是一个pb结构

```
message {
string user_name; // 目前仅有一个user_name字段，可以留作将来扩展
}
```

# 访问控制
表格ACL和用户信息持久化存储在meta表中。

这样带来一个问题：**性能**

解决方案：

1. master可以在内存中保存一份，每个RPC请求查内存数据就可以决定是否允许操作；
1. ts不能在每个RPC请求到达时查meta表，因为性能太差。

解决方案：

（表格ACL和用户信息）副本由master写入nexus或zk，可以预见数据量不会太大。
Ts对以上数据执行watch，在更新时向nexus或zk请求新数据，或者向master请求数据？

nexus/zk中存储：

1. acl目录下有一个叶子结点user，存储所有用户信息；
1. acl目录下再有一个tables目录结点，其下很多子结点，每个table对应一个叶子结点，值为该表格的schema

例如：当一个write的RPC到达ts，ts从rpc中取出token，对比内存中的表格schema.
