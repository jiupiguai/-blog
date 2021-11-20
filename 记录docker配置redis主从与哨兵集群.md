**话不多说 先上代码**

 **docker-compose.yml ：**
 ```yaml
# redis 主从复制+哨兵集群
version: '3.3'
services:
  sentinel-1:
    environment:
      - TZ=Asia/Shanghai # 设置时区
    image: "redis:latest" #redis 镜像源
    container_name: sentinel-1   # 容器的名字
    #  --restart=always参数能够使我们在重启docker时，自动启动相关容器。
    # Docker容器的重启策略如下：
    # no，默认策略，在容器退出时不重启容器
    # on-failure，在容器非正常退出时（退出状态非0），才会重启容器
    # on-failure:3，在容器非正常退出时重启容器，最多重启3次
    # always，在容器退出时总是重启容器
    # unless-stopped，在容器退出时总是重启容器，但是不考虑在Docker守护进程启动时就已经停止了的容器
    restart: always
    # 哨兵启动命令  配置文件的路径为  /usr/local/etc/redis/conf/sentinel.conf  -
    # -- 通过下面的volumes 的命令将宿主机中的配置文件映射到该路径
    command: redis-sentinel /usr/local/etc/redis/conf/sentinel.conf
    ports:
      - 26379:26379 # 将宿主机的26379端口 与Docker容器的26379进行绑定  宿主机端口：docker容器端口
    volumes:
      - ./sentinel.conf:/usr/local/etc/redis/conf/sentinel.conf  # 将宿主机的文件映射到docker容器内部   宿主机路径：docker内路径

  sentinel-2:
    environment:
      - TZ=Asia/Shanghai
    image: "redis:latest"
    container_name: sentinel-2
    restart: always
    command: redis-sentinel /usr/local/etc/redis/conf/sentinel.conf
    ports:
      - 26380:26379
    volumes:
      - ./sentinel.conf:/usr/local/etc/redis/conf/sentinel.conf

  sentinel-3:
    environment:
      - TZ=Asia/Shanghai
    image: "redis:latest"
    container_name: sentinel-3
    restart: always
    command: redis-sentinel /usr/local/etc/redis/conf/sentinel.conf
    ports:
      - 26381:26379
    volumes:
      - ./sentinel.conf:/usr/local/etc/redis/conf/sentinel.conf

  redie-1:
    environment:
      - TZ=Asia/Shanghai
    image: "redis:latest"
    container_name: redis-1
    restart: always
    ports:
      - 6379:6379
    volumes:
      - /root/docker/redis/6379-data:/data #  将redis持久化的数据文件映射到磁盘
      - ./redis.conf:/etc/redis.conf
    command:
       ["redis-server","/etc/redis.conf","--port","6379" ]  # 用--port 指定端口的的话 配置文件就不要每个都去改端口了



  redie-2:
    environment:
      - TZ=Asia/Shanghai
    image: "redis:latest"
    container_name: redis-2
    restart: always
    ports:
      - 6380:6380
    command:
      [ "redis-server","/etc/redis.conf","--port","6380" ]
    volumes:
      - /root/docker/redis/6380-data:/data
      - ./redis-slever.conf:/etc/redis.conf

  redie-3:
    environment:
      - TZ=Asia/Shanghai
    image: "redis:latest"
    container_name: redis-3
    restart: always
    ports:
      - 6381:6381
    command:
      [ "redis-server","/etc/redis.conf","--port","6381" ]
    volumes:
      - /root/docker/redis/6381-data:/data
      - ./redis-slever.conf:/etc/redis.conf
 ```
**redis.conf：**
```conf
	deamonize no
	 # 之前没加这行配置的时候 出现 master断开重连后 身份降级为slave
	 #  重新连接新的master的时候无法通过auth验证  
	masterauth password   
	#	 密码
	reduirepass password   
	# 开启aof持久化
	appendonly yes
	
```

**redis-selver.conf:**

```conf
	deamonize no
	masterauth password   
	#	 密码
	reduirepass password   
	# 开启aof持久化
	appendonly yes
	# 标记这个redis为 host的slave  host 是master 的ip地址
	replicaof [host] 6379 
```

**sentinel.conf:**
```conf
	# redis 因为都单独的部署在docker容器内 所以端口都配置为默认的26379没有做改动
	port:26379   
	deamonize no
	# sentinel monitor <master-name> <ip> <redis-port> <quorum>
	# quorum 为redis哨兵投票最小数 一般过半就可以 当master挂掉之后 哨兵会通过投票来开始failover
	sentinel monitor mymaster ip 6379 2
	# 链接redis 的密码
	sentinel auth-pass mymaster password
	sentinel down-after-milliseconds mymaster 30000
	requirepass password



```

**这是sentinel monitor <master-name> <ip> <redis-port> <quorum> 的官方解释：**
```text
sentinel monitor <master-name> <ip> <redis-port> <quorum> 
# 
# Tells Sentinel to monitor this master, and to consider it in O_DOWN 
# (Objectively Down) state only if at least <quorum> sentinels agree. 
# 
# Note that whatever is the ODOWN quorum, a Sentinel will require to 
# be elected by the majority of the known Sentinels in order to 
# start a failover, so no failover can be performed in minority. 
# 
# Replicas are auto-discovered, so you don't need to specify replicas in 
# any way.  Sentinel itself will rewrite this configuration file adding 
# the replicas using additional configuration options. 
# Also note that the configuration file is rewritten when a 
# replica is promoted to master. 
# 
# Note: master name should not include special characters or spaces. 
# The valid charset is A-z 0-9 and the three characters ".-_".
```

启动：
![在这里插入图片描述](https://img-blog.csdnimg.cn/7e76594d85ff48fbafd0586718c84484.png)
用redis-cli 进入主节点 查看信息：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2fe6f4d469b640a9825423d641671eba.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA6IKJ5pyr6IyE5a2QJg==,size_20,color_FFFFFF,t_70,g_se,x_16)
节点role为master ，接着有两个slave 说明链接正确 ，我们现在shut down主节点

从节点已经开始报错 无法连接到主节点
![在这里插入图片描述](https://img-blog.csdnimg.cn/a18ca4273ec1468a90cf7212e50d5b6d.png)
这时候半数哨兵判断master已经下线（客观判断）
![在这里插入图片描述](https://img-blog.csdnimg.cn/5967570e408b49dcb561f45085bd24dd.png)

哨兵选出新的master 并且将原先的slave转移到新的master下
.![在这里插入图片描述](https://img-blog.csdnimg.cn/509af9c153094729a4094d7fa3e33bda.png)

这时候 我们重启原先的master：
![在这里插入图片描述](https://img-blog.csdnimg.cn/cfd657aaef5d47588bfe77738c2285b7.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA6IKJ5pyr6IyE5a2QJg==,size_18,color_FFFFFF,t_70,g_se,x_16)
这时候 可以看到 这个节点 的role已经从master降到slave了  并且他的master是哨兵新选举出来的节点  说明配置成功