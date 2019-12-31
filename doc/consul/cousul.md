#Consul集群搭建

### 1.1 Consul镜像拉取

```
docker pull consul:1.4.4
```

### 2.2 Consul Server实例创建

　　以下我的实践是在一台机器上（CentOS 7）操作的，因此将三个实例分别使用了不同的端口号（区别于默认端口号8500）。实际环境中，建议多台机器部署。

#####（1）Consul实例1

```
sudo docker run -d -p 8510:8500 --restart=always -v /XiLife/consul/data/server1:/consul/data -v /XiLife/consul/conf/server1:/consul/config -e CONSUL_BIND_INTERFACE='eth0' --privileged=true --name=consul_server_1 consul:1.4.4 agent -server -bootstrap-expect=3 -ui -node=consul_server_1 -client='0.0.0.0' -data-dir /consul/data -config-dir /consul/config -datacenter=xdp_dc;
```