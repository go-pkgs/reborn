## Reborn 使用文档

Reborn 是一个分布式 Redis 解决方案, 对于上层的应用来说, 连接到 Reborn Proxy 和连接原生的 Redis Server 没有明显的区别 (不支持的命令列表), 上层应用可以像使用单机的 Redis 一样使用, Reborn 底层会处理请求的转发, 不停机的数据迁移等工作, 所有后边的一切事情, 对于前面的客户端来说是透明的, 可以简单的认为后边连接的是一个内存无限大的 Redis 服务.

Reborn 由四部分组成:

* Reborn Proxy   (reborn-proxy)
* Reborn Manager (reborn-config)
* Reborn Redis   (reborn-server)
* ZooKeeper

reborn-proxy 是客户端连接的 Redis 代理服务, reborn-proxy 本身实现了 Redis 协议, 表现得和一个原生的 Redis 没什么区别 (就像 Twemproxy), 对于一个业务来说, 可以部署多个 reborn-proxy, reborn-proxy 本身是无状态的.

reborn-config 是 Reborn 的管理工具, 支持包括, 添加/删除 Redis 节点, 添加/删除 Proxy 节点, 发起数据迁移等操作. reborn-config 本身还自带了一个 http server, 会启动一个 dashboard, 用户可以直接在浏览器上观察 Reborn 集群的运行状态.

reborn-server 是 Reborn 项目维护的一个 Redis 分支, 基于 2.8.13 开发, 加入了 slot 的支持和原子的数据迁移指令. Reborn 上层的 reborn-proxy 和 reborn-config 只能和这个版本的 Redis 交互才能正常运行.

Reborn 依赖 ZooKeeper 来存放数据路由表和 reborn-proxy 节点的元信息, reborn-config 发起的命令都会通过 ZooKeeper 同步到各个存活的 reborn-proxy.

Reborn 支持按照 Namespace 区分不同的产品, 拥有不同的 product name 的产品, 各项配置都不会冲突.


###Build reborn-proxy & reborn-config
------------------

安装go[参考这里](https://golang.org/doc/install)，建议使用Go源码安装，然后参考下的流程

```
go get github.com/reborndb/reborn
cd $GOPATH/src/github.com/reborndb/reborn
./bootstrap.sh
make gotest
```

会在 reborn/bin 文件夹生成 reborn-config, reborn-proxy 两个可执行文件, (另外, bin/assets 文件夹是 reborn-config 的 dashboard http 服务需要的前端资源, 需要和 reborn-config 放置在同一文件夹下)

```
cd sample

$ ../bin/reborn-config -h                                                                                                                                                                                                                           (master)
usage: reborn-config  [-c <config_file>] [-L <log_file>] [--log-level=<loglevel>]
		<command> [<args>...]
options:
   -c	配置文件地址
   -L	日志输出文件地址
   --log-level=<loglevel>	输出日志级别 (debug < info (default) < warn < error < fatal)

commands:
	server            redis 服务器组管理
	slot              slot 管理
	dashboard         启动 dashboard 服务
	action            事件管理 (目前只有删除历史事件的日志)
	proxy             proxy 管理
```

```
$ ../bin/reborn-proxy -h

usage: reborn-proxy [options]

options:
   -c	配置文件地址
   -L	日志输出文件地址
   --log-level=<loglevel>      输出日志级别 (debug < info (default) < warn < error < fatal)
   --cpu=<cpu_num>             proxy占用的 cpu 核数, 默认1, 最好设置为机器的物理cpu数的一半到2/3左右
   --addr=<proxy_listen_addr>  proxy 的 redis server 监听的地址, 格式 <ip or hostname>:<port>, 如: localhost:9000, :9001
   --id=<proxy_id>             proxy id，需要全局唯一，不能为空
   --http-addr=<debug_http_addr>   proxy 的调试信息启动的http server, 可以访问 http://debug_http_server_addr/debug/vars
```

###部署
------------------------

####配置文件

reborn-config 和 reborn-proxy 在不加 -c 参数的时候, 默认会读取当前目录下的 config.ini 文件

config.ini:

```
coordinator_addr=localhost:2181   <- zookeeper的地址, 如果是zookeeper集群，可以这么写: coordinator_addr=hostname1:2181,hostname2:2181,hostname3:2181,hostname4:2181,hostname5:2181
如果是etcd，则写成http://hostname1:port,http://hostname2:port,http://hostname3:port
product=test        <- 产品名称, 这个reborn集群的名字, 可以认为是命名空间, 不同命名空间的reborn没有交集
dashboard_addr=localhost:18087   <- dashboard 服务的地址，CLI 的所有命令都依赖于 dashboard 的 RESTful API，所以必须启动
coordinator=zookeeper  <- 如果用etcd，则将zookeeper替换为etcd
```

####流程

**0. 启动 dashboard**, 执行 `../bin/reborn-config dashboard`, 该命令会启动 dashboard

**1. 初始化 slots** , 执行 `../bin/reborn-config slot init`，该命令会在zookeeper上创建slot相关信息

**2. 启动 Reborn Redis** , 和官方的Redis Server参数一样

**3. 添加 Redis Server Group** , 每一个 Server Group 作为一个 Redis 服务器组存在, 只允许有一个 master, 可以有多个 slave, ***group id 仅支持大于等于1的整数***

```
$ ../bin/reborn-config server -h                                                                                                                                                                                                                   usage:
	reborn-config server list
	reborn-config server add <group_id> <redis_addr> <role>
	reborn-config server remove <group_id> <redis_addr>
	reborn-config server promote <group_id> <redis_addr>
	reborn-config server add-group <group_id>
	reborn-config server remove-group <group_id>
```
如: 添加两个 server group, 每个 group 有两个 redis 实例，group的id分别为1和2，
redis实例为一主一从。

添加一个group，group的id为1， 并添加一个redis master到该group
```
$ ../bin/reborn-config server add 1 localhost:6379 master
```
添加一个redis slave到该group
```
$ ../bin/reborn-config server add 1 localhost:6380 slave
```
类似的，再添加group，group的id为2
```
$ ../bin/reborn-config server add 2 localhost:6479 master
$ ../bin/reborn-config server add 2 localhost:6480 slave
```

**4. 设置 server group 服务的 slot 范围**
   Reborn 采用 Pre-sharding 的技术来实现数据的分片, 默认分成 1024 个 slots (0-1023), 对于每个key来说, 通过以下公式确定所属的 Slot Id : SlotId = crc32(key) % 1024 
   每一个 slot 都会有一个特定的 server group id 来表示这个 slot 的数据由哪个 server group 来提供.

```
$ ../bin/reborn-config slot -h                                                                                                                                                                                                                     
usage:
	reborn-config slot init
	reborn-config slot info <slot_id>
	reborn-config slot set <slot_id> <group_id> <status>
	reborn-config slot range-set <slot_from> <slot_to> <group_id> <status>
	reborn-config slot migrate <slot_from> <slot_to> <group_id> [--delay=<delay_time_in_ms>]
```

如: 

设置编号为[0, 511]的 slot 由 server group 1 提供服务, 编号 [512, 1023] 的 slot 由 server group 2 提供服务

```
$ ../bin/reborn-config slot range-set 0 511 1 online
$ ../bin/reborn-config slot range-set 512 1023 2 online
```

 **5. 启动 reborn-proxy**
```
 ../bin/reborn-proxy -c config.ini -L ./log/proxy.log  --cpu=8 --addr=0.0.0.0:19000 --http-addr=0.0.0.0:11000
```
刚启动的 reborn-proxy 默认是处于 offline状态的, 然后设置 proxy 为 online 状态, 只有处于 online 状态的 proxy 才会对外提供服务
```
 ../bin/reborn-config -c config.ini proxy online <proxy_name>  <---- proxy的id, 如 proxy_1
```

 **6. 打开浏览器 http://localhost:18087/admin**
 
 现在可以在浏览器里面完成各种操作了， 玩得开心
  

###数据迁移
-----------------------------

安全和透明的数据迁移是 Reborn 提供的一个重要的服务, 也是 Reborn 区别于 Twemproxy 等静态的分布式 Redis 解决方案的地方.

数据迁移的最小单位是 key, 我们在 reborn redis 中添加了一些指令, 实现基于key的迁移, 如 SLOTSMGRT等 (命令列表),  每次会将特定 slot 一个随机的 key 发送给另外一个 reborn redis 实例, 这个命令会确认对方已经接收, 同时删除本地的这个  k-v 键值, 返回这个  slot 的剩余 key 的数量, 整个操作是原子的.

在 reborn-config 管理工具中, 每次迁移任务的最小单位是 slot

如: 将slot id 为 [0-511] 的slot的数据, 迁移到 server group 2上,  --delay 参数表示每迁移一个 key 后 sleep 的毫秒数, 默认是 0, 用于限速.

```
$ ../bin/reborn-config slot migrate 0 511 2 --delay=10
```

迁移的过程对于上层业务来说是安全且透明的, 数据不会丢失,  上层不会中止服务.

注意, 迁移的过程中打断是可以的, 但是如果中断了一个正在迁移某个slot的任务, 下次需要先迁移掉正处于迁移状态的 slot, 否则无法继续 (即迁移程序会检查同一时刻只能有一个 slot 处于迁移状态).


####Auto Rebalance 

Reborn 支持动态的根据实例内存, 自动对slot进行迁移, 以均衡数据分布.

```
$ ../bin/reborn-config slot rebalance
```

要求:
 * 所有的reborn-server都必须设置了maxmemory参数
 * 所有的 slots 都应该处于 online 状态, 即没有迁移任务正在执行
 * 所有 server group 都必须有 Master
 * 

####如实现reborn-server的主从自动切换

reborn-ha是一个通过reborn开放的api实现自动切换主从的例子。[具体用法](https://github.com/ngaut/reborn-ha)
