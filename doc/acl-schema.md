# Root用户不受访问控制的任何约束

# 表格的访问控制信息在表格的schema中
```
TableSchema {
String owner;
String group;
String acl; // R：读， W：写， A：管理（delete、disable、drop、update）
}
```
例如：
```
Linkbase {
owner :junyi
group :linkbase-user
acl :RWARW-R--    // owner有权限读、写、管理；group有权限读、写；其它用户可以读
}
```
# 用户信息存在meta表中
每个用户都对应一个唯一的token，这个token会存在于RPC请求中，token用于决定用户身份（此RPC请求是张三还是李四发起的）。

**Key**: ~token

**Value**: 是一个pb结构

```
message {
string user_name;
repeated string group_name;
string acl;  //是否可以在集群中建表（每次建表都需要root是否太麻烦），更多权限可扩展
}
```

# 访问控制
表格ACL和用户信息持久化存储在meta表中
；
这样带来一个问题：**性能**

解决方案：
1. master可以在内存中保存一份，每个RPC请求查内存数据就可以决定是否允许操作；
1. ts不能在每个RPC请求到达时查meta表，因为性能太差。ts可以读一次meta表，然后缓存下来。

这样又带来一个问题：**缓存的更新与一致性**

（表格ACL和用户信息）副本由master写入nexus或zk，可以预见数据量不会太大。
Ts对以上数据执行watch，只有在更新时才会重新访问meta表，或者向master请求数据？

例如：当一个write的RPC到达ts，ts从rpc中取出token，对比内存中的表格schema.
