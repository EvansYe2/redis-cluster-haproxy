## 基于Docker构建Redis-Cluster及Haproxy（懒人版）


### 如何使用

1. clone项目源码
   运行命令之前，需要将generate.sh里面的ip地址改成您当前宿主机的ip地址(示例里面是内网地址)
运行命令：

  ```
  sh generate.sh
  ```
示例里面redis采用的端口范围：6380-6385，具体可以看每个文件夹下面的redis.conf


2. 生成节点对应的data和redis.conf文件，同时修改haproxy.cfg增加如下图的代码：
![文件夹结构](https://github.com/EvansYe2/redis-cluster-haproxy-generate/blob/main/imgs/files.png?raw=true)

![haproxy修改](https://github.com/EvansYe2/redis-cluster-haproxy-generate/blob/main/imgs/haproxy-redis.png?raw=true)
  ```
  注意：如每次重新运行sh generate.sh，一定要先删除之前生成的文件夹，以及haproxy.cfg文件里面新增的内容
  ``` 

3. 构建集群
  命令里面的ip改成自己宿主机的ip，构建3主3从
  ```
  $ docker exec -it redis-6380  redis-cli -h 127.0.0.1 -p 6380 -a 123456789 --cluster create 192.168.31.108:6380 192.168.31.108:6381 192.168.31.108:6382 192.168.31.108:6383 192.168.31.108:6384 192.168.31.108:6385 --cluster-replicas 1 --cluster-yes
  
  ```
 --cluster create 表⽰建⽴集群.后⾯填写每个节点的ip和地址.
 --cluster-replicas 1 表⽰每个主节点需要一个从节点备份.

执⾏之后,容器之间会进⾏加⼊集群操作.
⽇志中会描述哪些是主节点,哪些从节点跟随哪个主节点.
![创建集群](https://github.com/EvansYe2/redis-cluster-haproxy-generate/blob/main/imgs/create-cluster.png?raw=true)

此时,使⽤客⼾端连上集群中的任何⼀个节点,都相当于连上了整个集群.

客⼾端后⾯要加上-c 选项,否则如果key没有落到当前节点上,是不能操作的. -c 会自动吧请求重定向到对应节点.
使⽤ cluster nodes 可以查看到整个集群的情况.

4. 验证集群：
info replication 查看Redis节点信息
cluster nodes 查看集群节点信息

  ```
docker exec -it redis-6380 redis-cli -h 127.0.0.1 -p 6380 -a 123456789 info replication

docker exec -it redis-6380 redis-cli -h 127.0.0.1 -p 6380 -a 123456789 cluster nodes
  ```

### redis-cli -c 客户端使用集群模式

1.写入数据到节点:

```
docker exec -it redis-6380 redis-cli -c -h 127.0.0.1 -p 6380 -a 123456789 SET name1 "test_cluster_name1"

docker exec -it redis-6381 redis-cli -c -h 127.0.0.1 -p 6381 -a 123456789 SET name2 "test_cluster_name2"

docker exec -it redis-6382 redis-cli -c -h 127.0.0.1 -p 6382 -a 123456789 SET name3 "test_cluster_name3"

```
 
2. 节点读取数据查看是否设置成功

  ```
  $ docker exec -it redis-6385 redis-cli -c -h 127.0.0.1 -p 6385 -a 123456789 GET name1
  $ docker exec -it redis-6384 redis-cli -c -h 127.0.0.1 -p 6384 -a 123456789 GET name2
  $ docker exec -it redis-6383 redis-cli -c -h 127.0.0.1 -p 6383 -a 123456789 GET name3
  ```

3.工具连接任意一个节点：
![连接任意一个节点](https://github.com/EvansYe2/redis-cluster-haproxy-generate/blob/main/imgs/%E7%9B%B4%E6%8E%A5%E8%BF%9E%E6%8E%A5%E4%BB%BB%E6%84%8F%E4%B8%80%E4%B8%AA%E8%8A%82%E7%82%B9.png?raw=true)

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
![连接haproxy](https://github.com/EvansYe2/redis-cluster-haproxy-generate/blob/main/imgs/haproxy-connect-redis.png?raw=true)


2.HAProxy 的状态页（Stats Page） 是实时监控负载均衡集群的核心工具，通过 Web 页面展示关键性能指标和后端节点状态：
http://localhost:8080/haproxy
账户密码在：haproxy.cfg文件里面
![HAProxy 的状态页](https://github.com/EvansYe2/redis-cluster-haproxy-generate/blob/main/imgs/haproxy.png?raw=true)

补充：连接redis用的工具是：https://github.com/qishibo/AnotherRedisDesktopManager

redis官方也有免费的管理工具：https://redis.io/insight/

Enjoy.
