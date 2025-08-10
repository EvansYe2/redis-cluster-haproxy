## 基于Docker构建Redis-Cluster及Haproxy（懒人版）


### 如何使用

1. Docker Compose简介
   

Docker Compose是一个用于定义和运行多容器Docker应用程序的工具。通过一个YAML文件，可以轻松地配置应用程序的服务、网络和卷等资源，并一键启动整个应用环境。


2. Redis Cluster简介
Redis Cluster是Redis官方提供的分布式解决方案，支持数据自动分片、节点间通信、故障转移等功能。Redis Cluster通过哈希槽（hash slot）的方式管理数据，共有16384个哈希槽，每个键根据CRC16算法分配到相应的槽中。

虚拟槽分区巧妙地使用了哈希空间，使用分散度良好的哈希函数把所有的数据映射到一个固定范围内的整数集合，整数定义为槽（slot）。比如：Redis Cluster槽的范围是0 ～ 16383。槽是集群内数据管理和迁移的基本单位。
Redis Cluster采用虚拟槽分区，所有的键根据哈希函数映射到0 ～ 16383，计算公式：slot = CRC16(key)&16383。每一个节点负责维护一部分槽以及槽所映射的键值数据。


3. 开始使用，clone项目源码

 
4. 修改cluster-announce-ip

  ```
  将docker-compose.yml 里面的 cluster-announce-ip  全部都改成宿主机ip，我这里用的是内网ip

  公用一个redis.conf,redis.conf无需做任何修改

  ```

5. 修改haproxy.cfg

  ```
  将listen  redis部分如下几个redis监听里面的ip改成跟步骤4一样的，全部都改成宿主机ip，我这里用的是内网ip

  server redis-master1 192.168.31.108:6379 check
  server redis-master2 192.168.31.108:6380 check
  server redis-master3 192.168.31.108:6381 check
  server redis-slave1 192.168.31.108:6382 check
  server redis-slave2 192.168.31.108:6383 check
  server redis-slave3 192.168.31.108:6384 check

  ```

6. 执行yml文件，启动容器

  ```
  docker-compose up -d

  ```
![启动容器](https://github.com/EvansYe2/redis-cluster-haproxy/blob/main/imgs/start-docker.png?raw=true)


7. 构建集群
  命令里面的ip改成自己宿主机的ip，构建3主3从
  ```
  $ docker exec -it redis-master1  redis-cli -h 127.0.0.1 -p 6379 -a 123456789 --cluster create 192.168.31.108:6379 192.168.31.108:6380 192.168.31.108:6381 192.168.31.108:6382 192.168.31.108:6383 192.168.31.108:6384 --cluster-replicas 1 --cluster-yes
  
  ```
 --cluster create 表⽰建⽴集群.后⾯填写每个节点的ip和地址.
 --cluster-replicas 1 表⽰每个主节点需要一个从节点备份.

执⾏之后,容器之间会进⾏加⼊集群操作.
⽇志中会描述哪些是主节点,哪些从节点跟随哪个主节点.
![创建集群](https://github.com/EvansYe2/redis-cluster-haproxy/blob/main/imgs/create-cluster.png?raw=true)

此时,使⽤客⼾端连上集群中的任何⼀个节点,都相当于连上了整个集群.

客⼾端后⾯要加上-c 选项,否则如果key没有落到当前节点上,是不能操作的. -c 会自动吧请求重定向到对应节点.
使⽤ cluster nodes 可以查看到整个集群的情况.

8. 验证集群：
info replication 查看Redis节点信息
cluster nodes 查看集群节点信息

  ```

  docker exec -it redis-master1 redis-cli -h 127.0.0.1 -p 6379 -a 123456789 info replication

  docker exec -it redis-master1 redis-cli -h 127.0.0.1 -p 6379 -a 123456789 cluster nodes
  
  ```

### redis-cli -c 客户端使用集群模式

1.写入数据到节点:

```
docker exec -it redis-master1 redis-cli -c -h 127.0.0.1 -p 6379 -a 123456789 SET name "test_cluster_name"

```
 
2. 节点读取数据查看是否设置成功

  ```
  $ docker exec -it redis-master2 redis-cli -c -h 127.0.0.1 -p 6379 -a 123456789 GET name

  $ docker exec -it redis-slave1 redis-cli -c -h 127.0.0.1 -p 6379 -a 123456789 GET name

  $ docker exec -it redis-6383 redis-cli -c -h 127.0.0.1 -p 6383 -a 123456789 GET name3

  ```

3.工具连接任意一个节点：
![连接任意一个节点](https://github.com/EvansYe2/redis-cluster-haproxy/blob/main/imgs/%E7%9B%B4%E6%8E%A5%E8%BF%9E%E6%8E%A5%E4%BB%BB%E6%84%8F%E4%B8%80%E4%B8%AA%E8%8A%82%E7%82%B9.png?raw=true)

### 通过HAProxy连接
是法国开发者 威利塔罗(Willy Tarreau) 在2000年使用C语言开发的一个开源软件
是一款具备高并发(万级以上)、高性能的TCP和HTTP负载均衡器
支持基于cookie的持久性，自动故障切换，支持正则表达式及web状态统计
企业版网站：https://www.haproxy.com
社区版网站：http://www.haproxy.org
github：https://github.com/haprox
核心功能： 
负载均衡（Load Balancing）
支持四层（TCP）和七层（HTTP/HTTPS）流量分发。
提供多种调度算法：轮询（roundrobin）、最少连接（leastconn）、源IP哈希（source）等。
反向代理（Reverse Proxy）

隐藏后端服务器细节，对外提供统一入口。
支持 SSL 终端（SSL Termination），卸载后端服务器加密负担。
高可用（High Availability）

结合 Keepalived 实现双机热备（VRRP 协议）。
流量治理

请求过滤、速率限制、连接控制等。
haproxy特点和优点：
支持原生SSL,同时支持客户端和服务器的SSL.
支持IPv6和UNIX套字节（sockets）
支持HTTP Keep-Alive
支持HTTP/1.1压缩，节省宽带
支持优化健康检测机制（SSL、scripted TCP、check agent…）
支持7层负载均衡。
可靠性和稳定性非常好。
并发连接 40000-50000个，单位时间处理最大请求 20000个，最大数据处理10Gbps.
支持8种负载均衡算法，同时支持session保持。
支持虚拟主机。
支持连接拒绝、全透明代理。
拥有服务器状态监控页面。
支持ACL（access control list）。

1.连接haproxy，haproxy会代理转发：
![连接haproxy](https://github.com/EvansYe2/redis-cluster-haproxy/blob/main/imgs/haproxy-connect-redis.png?raw=true)


2.HAProxy 的状态页（Stats Page） 是实时监控负载均衡集群的核心工具，通过 Web 页面展示关键性能指标和后端节点状态：
http://localhost:8080/haproxy
账户密码在：haproxy.cfg文件里面
![HAProxy 的状态页](https://github.com/EvansYe2/redis-cluster-haproxy/blob/main/imgs/haproxy.png?raw=true)

补充：连接redis用的工具是：https://github.com/qishibo/AnotherRedisDesktopManager

redis官方也有免费的管理工具：https://redis.io/insight/

Enjoy.
