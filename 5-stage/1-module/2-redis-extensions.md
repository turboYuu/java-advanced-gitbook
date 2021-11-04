第二部分 Redis扩展功能

# 5 发布与订阅

# 6 事务

# 7 Lua脚本

# 8 慢日志查询

```yaml
# 执行时间超过多少微秒的命令请求会被记录到日志上 0 :全记录 <0 不记录 
slowlog-log-slower-than  10000
#slowlog-max-len 存储慢查询日志条数 
slowlog-max-len 128
```

临时设置

config set

```shell
127.0.0.1:6379> config set slowlog-log-slower-than 0
OK
127.0.0.1:6379> config set slowlog-max-len 5
OK
127.0.0.1:6379> slowlog get 1
1) 1) (integer) 2
   2) (integer) 1636018486
   3) (integer) 3
   4) 1) "slowlog"
      2) "get"
      3) "[1]"
   5) "127.0.0.1:55990"
   6) ""
127.0.0.1:6379> slowlog get 3
1) 1) (integer) 3
   2) (integer) 1636018500
   3) (integer) 16
   4) 1) "slowlog"
      2) "get"
      3) "1"
   5) "127.0.0.1:55990"
   6) ""
2) 1) (integer) 2
   2) (integer) 1636018486
   3) (integer) 3
   4) 1) "slowlog"
      2) "get"
      3) "[1]"
   5) "127.0.0.1:55990"
   6) ""
3) 1) (integer) 1
   2) (integer) 1636018472
   3) (integer) 4
   4) 1) "config"
      2) "set"
      3) "slowlog-max-len"
      4) "5"
   5) "127.0.0.1:55990"
   6) ""
127.0.0.1:6379> 
```



# 9 监视器

Redis客户端1

```shell
127.0.0.1:6379> monitor
OK
1636019353.766249 [0 127.0.0.1:55992] "COMMAND"
1636019375.414681 [0 127.0.0.1:55992] "set" "name" "turbo"
1636019380.011170 [0 127.0.0.1:55992] "set" "name" "echo"
```

Redis客户端2

```shell
127.0.0.1:6379> set name turbo
OK
127.0.0.1:6379> set name echo
OK
```

