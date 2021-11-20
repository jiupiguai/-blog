在redis集群中 地址发现会失败 ，项目中明明配置的是公网的ip，可是从集群获取信息后转换成了内网ip 开始以为是项目的配置问题，经过多次调试，确认项目配置无误，于是开始redis排查 最终发现如下问题：
```conf
# In certain deployments, Redis Cluster nodes address discovery fails, because 
# addresses are NAT-ted or because ports are forwarded (the typical case is 
# Docker and other containers). 
# 
# In order to make Redis Cluster working in such environments, a static 
# configuration where each node knows its public address is needed.  The 
# following two options are used for this scope, and are: 
# 
# * cluster-announce-ip 
# * cluster-announce-port 
# * cluster-announce-bus-port 
# 
# Each instruct the node about its address, client port, and cluster message 
# bus port.  The information is then published in the header of the bus packets 
# so that other nodes will be able to correctly map the address of the node 
# publishing the information. 
# 
# If the above options are not used, the normal Redis Cluster auto-detection 
# will be used instead. 
# 
# Note that when remapped, the bus port may not be at the fixed offset of 
# clients port + 10000, so you can specify any port and bus-port depending 
# on how they get remapped.  If the bus-port is not set, a fixed offset of 
# 10000 will be used as usually. 
# 
# Example: 
# 
# cluster-announce-ip 10.1.1.5 
# cluster-announce-port 6379 
# cluster-announce-bus-port 6380
```
这是redis.conf 中的原文 大致意思是在docker或者某些部署中 因为端口被转发等其他原因 会导致地址发送变化，所以在配置中写死地址就可以解决 