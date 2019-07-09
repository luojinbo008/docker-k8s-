#k8s 通过kubeadm 管理工具配置集群
kubeadm([文档链接](https://github.com/kubernetes/kubeadm/blob/master/docs/design/design_v1.10.md))

步骤:
###1.  master，node:安装kubelet，kubeadm，docker_ce（[docker安装](docker-install.md)) 
  
    准备：
  
    如果各个主机启用了防火墙，需要开放Kubernetes各个组件所需要的端口。 这里简单起见在各节点禁用防火墙：
       
        systemctl disable firewalld 
        systemctl disable iptables
            
    禁用SELINUX：(程序操作文件权限。。。和用户（root无关)）
    
         setenforce 0
    
         vi /etc/selinux/config
         SELINUX=disabled
         
    确保 服务器hostname 合法规则
        
         hostname
         
   阿里云 yum源准备 
   
        cd /etc/yum.repos.d/
        vim kubernetes.repo
    
        # kubernetes.repo file
        
            
        [kubernetes]
        name=Kubernetes
        baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
        enabled=1
        gpgcheck=0
        repo_gpgcheck=0
        gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
               http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
           
   yum 源查看
   
        yum repolist
    
   ![yum repolist](../../doc-img/k8s/1.png)
   
   
   安装 docker_ce, kubeadm, kubelet (master, node 都需要安装)
  
    yum install docker_ce kubeadm kubelet
  
   被墙因素操作 设置代理（或其他方式科学上网方式解决）词方法废弃，本人利用阿里云制作国内镜像解决
  
       vim /usr/lib/systemd/system/docker.service 
       
       目前已经失效，寻找新代理
       Environment="HTTPS_PROXY=http://www.ik8s.io:10080" 
       Environment="NO_PROXY=127.0.0.0/8, 172.20.0.0/16"
       
   
   
   下拉需要的images（采用阿里云制作国内镜像，在修改本地镜像名称）
       
       // sh 文件 方便操作 并执行sh文件
        
       #!/bin/bash
       images=(kube-apiserver:v1.15.0 kube-controller-manager:v1.15.0 kube-scheduler:v1.15.0 kube-proxy:v1.15.0 pause:3.1 etcd:3.3.10 coredns:1.3.1)
       for imageName in ${images[@]} ; do
       docker pull registry.cn-hangzhou.aliyuncs.com/dumb008/$imageName
       docker tag registry.cn-hangzhou.aliyuncs.com/dumb008/$imageName k8s.gcr.io/$imageName
       docker rmi registry.cn-hangzhou.aliyuncs.com/dumb008/$imageName
       done
                
       // 查看镜像
       
       docker images
        
   确保  kernel 开启 bridge-nf-call-iptables 和 bridge-nf-call-ip8tables 
   
       cat /proc/sys/net/bridge/bridge-nf-call-iptables 
       1
       cat /proc/sys/net/bridge/bridge-nf-call-ip6tables 
       1
       
   启动docker，kubeadm, kubelet
  
    开机自动启动
    systemctl enable docker.service
    systemctl enable kubelet
  
       
####2.  master:kubeadm init(初始化 集群初始化)


    // 初始化 master
    
    kubeadm init --kubernetes-version=v1.15.0 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 

    // 失败 重置 
    kubeadm reset
    
  
    // 配置  
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    
    // 其他用户启用
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    
    // 查看状态
    kubectl get cs
   
    // 部署pod 网络
    // You should now deploy a pod network to the cluster.
    
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/
    

    
    kubectl get nodes
    
![yum repolist](../../doc-img/k8s/2.png)

   node 没有准备好 需要安装 fannel
   
   
    // 安装 flannel
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    
    // 查看状态
    kubectl get nodes 
    
    // 查看 pods 和 名称空间
    kubectl get pods -n kube-system
    kubectl get ns

####3.  node：kubeadm join （node 加入集群）
    
    
    // node 加入集群
    Then you can join any number of worker nodes by running the following on each as root:
    
    kubeadm join 172.30.0.15:6443 --token u5047s.tipq90q9bnk8czg2 \
        --discovery-token-ca-cert-hash sha256:4b5d8242434204a3eed4d54b240498f0878e44dcd4666e082a133fecf82bac81




####4. kubectl（管理入口-客户端）


   kubectl: 链接 api-server 对各种资源对正删改查。。。
   
   资源：pods， server， controller（replicaset，deployment, daemonset, job, crontjob 控制器种类），node 



####5. k8s 命令创建环境

######1.   创建pods
   
    kubectl run 【name】--image=【image】--port=80 --relicas=【pods个数】 // 创建pods
    
    kubectl get pods -o wide
    
    
######2.   外部client 访问集群内部网络（通过特殊的service）
    
    // 资源包括(不区分大小写)：
    // pod（po），service（svc），replication controller（rc），deployment（deploy），replica set（rs）
    
    // 将资源暴露为新的Kubernetes Service。
    
    kubectl expose 资源类型【pods name】--name=【name】--port=80 --target-port=80 --protocol=TCP

    // 这一步说是将服务暴露出去，实际上是在服务前面加一个负载均衡，因为pod可能分布在不同的结点上。
     
    –port：暴露出去的端口 
    –type=NodePort：使用结点 + 端口方式访问服务 
    –target-port：容器的端口 
    –name：创建service指定的名称
    
    // 查看 service 信息
    kubectl get scv -n name
    
    // 集群外部访问，修改svc
    kubectl edit svc [name]
    // 其中spec.type=ClusterIP 改成 spec.type=NodePort 
    
    // 然后查看svc 信息
    kubectl get svc
    
####5 通过配置清单文件(yaml文件) 搭载 （推荐）
   
#####1.  k8s 核心资源: 对象
   
   workload（工作负载型资源）：pod, ReplicaSet, Deployment, DaemonSet, job, CrontJob 

   服务发现及服务均衡资源：Service，Ingress，...
   
   
   配置与存储型资源：Volume，CSI
    
  
   - ConfigMap（保存配置中心资源），Secret（保护敏感资源）
   
   - DownWardAPI （外部环境输出给容器的资源）
   
   
   集群级别到资源：
   
   - NameSpace, Node, Role, ClusterRole, RoleBinding, ClusterRoleBinding
   
   元数据资源:
   
   - HPA: PodTemplate, LimitTemplate
   
#####2.  搭载

例：获取一个pod 到配置

    kubectl get pod 【名称】-o yaml

    // key说明
    apiVersion group/version  group 默认 core（核心组
    
    kind pod 资源类别
    
    metaData 元数据
    
    spec 规格 定义资源对象应该有到特性（目标状态）
    
    status 当前状态 系统来维护
    
创建资源到方法：

- apiServer 仅接受 json 格式数据的资源定义

- yaml格式提供配置清单，apiServer 可自动转义成json，在执行

大部分资源的配置清单：

- apiVersion 以“组/版本”的格式输出服务端支持的API版本。
  
  命令 kubectl api-versions 
  
    
- kind 资源类别哦

- metaData 元数据

   - name 
    
   - namespace
   
   - labels 标签
   
   - annotation 备注

   - resourceVersion
   
   - uid
   
   - selfLink 每个资源的引用PATH      /api/GROUP/VERSION/namespaces/NAMESPACE/TYPE/NAME

- spec 期望的状态

- status 当前状态 （让当前状态无限接近 期望状态， 有k8s集群维护，用户无法定义）


k8s 格式说明

    kubectl explain pods | pods.metadata ...(资源)

创建

    kubectl create -f xxx.yaml
    
    

    
    
   
   
