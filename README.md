# swarm-doc
docker swarm doc &amp; example



## swarm架构

轻量级容器编排引擎，天然集成在docker中，不同于k8s的面向应用编排，swarm绑定docker，是一款面向docker容器编排工具，单个集群规模raft节点最大推荐7,worker数1000，容器数量50000

![img](https://ucc.alicdn.com/images/user-upload-01/img_convert/5015b26ce0a4e3f890467c469b45dee4.png)

## swarm raft高可用

保证存活Manager节点数量为(n-1)/2,节点会动态进行选举保证集群高可用,默认leader为集群init时指定的advertise-addr，leader故障时Reachable节点会发起选举，最快发起的节点会当选为新的raft leader，向Reachable节点发送指令会通过raft日志同步至leader节点，leader节点负责处理指令，下发至worker节点

swarm官方推荐Manager节点数量为3/5/7，数量过大因为raft算法同步过程较长会影响性能.

### swarm 节点端口

| 集群管理通讯tcp | 节点间通讯tcp | overlay服务网格udp |
| :-------------: | :-----------: | :----------------: |
|      2377       |     7946      |        4789        |

### overlay网络创建

```shell
# attachable
# live-restore为false

```

## swarm集群创建

```shell
# step1
docker swarm init --advertise-addr 172.22.54.97

# step2 创建overlay网络
docker network create --driver overlay --subnet=10.200.1.0/16 --attachable fabric

# step3 节点加入
# 获取manager节点加入命令
docker swarm join-token manager
# 获取worker节点加入命令
docker swarm join-token worker
# docker swarm join --token SWMTKN-1-61vuoifzk1fi2n5ycnt1ofkcpjt4gwa8y06mav118vxoj8kqmv-4u9321bjfjfch37og6vxsxrnk 172.22.54.97:2377
```

### swarm服务创建

```shell
docker stack -c my-stack.yaml my-app
```

```yaml
version: "3.3"
volumes:
  peer0.org1.example.com:
  peer1.org1.example.com:
networks:
  study-network:
    external:
      name: fabric
services:
  peer0_org1:
    deploy:
      replicas: 1 # 副本数量
      restart_policy: # 重启策略
        condition: on-failure 
        delay: 5s # 重启间隔
        max_attempts: 3 # 重启次数
      placement:
        constraints: # 定义部署约束,可以用hostname的方式选择部署到的节点
          - node.hostname == 10_245_150_86
    image: hyperledger/fabric-peer:$IMAGE_TAG
    hostname: peer0.org1.example.com
    ports:
      - 7051:7051
      - 7053:7053
    networks:
      study-network:
        aliases: # 容器在网络中的别名
          - peer0.org1.example.com

```

### swarm服务滚动更新

```yaml
version: '3.8'

services:
  web:
    image: nginx:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 2  # 每次更新替换的容器数量
        delay: 10s      # 更新任务之间的延迟时间
        failure_action: rollback  # 更新失败时的行为
        monitor: 30s    # 监控容器健康的时间间隔
```

### swarm服务扩缩容

```shell
docker service scale my-web=2 // 服务扩(缩)容
```

### swarm集群退出

```shell
docker swarm leave -f
```

### swarm节点管理命令

```shell
# 查看节点
docker node ls
# 删除节点
docker node rm xxxxxxxxxxxx 
# 升级worker为manager
docker node promote node01 
# manager节点降级为worker
docker node demote manager01 
# 更新节点
docker node update --availability drain manager
# node update: 更改节点状态
# --availability: 三种状态
#	  active: 正常
#   pause：挂起
#   drain：排除

# 1.排除(排除后manager只作为管理节点)
docker node update --availability drain master
# 2.允许
docker node update –availability active master
```

### swarm证书轮换

```shell
docker swarm ca --rotate
```







