# Kubernetes 集群部署

## 目标

在 三台服务器上搭建 一主两从 

## 1 事前准备

```shell
#同步时间：
yum install ntpdate -y
ntpdate ntp.api.bz

#修改三台服务器主机名
hostnamectl set-hostname xxxxx
hostnamectl status

#设置 hostname 解析
echo "127.0.0.1 $(hostname)" >> /etc/hosts
vim /etc/sysconfig/network
HOSTNAME=xxxx
vim /etc/hostname
k8s-master   k8s-node1   k8s-node2 (例)

#修改主机DNS
vim /etc/resolv.conf
nameserver 114.114.114.114
nameserver 8.8.8.8

#关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

#关闭selinux：
sed -i 's/enforcing/disabled/' /etc/selinux/config 
setenforce 0

#关闭swap：
swapoff -a $ 临时
vim /etc/fstab $ 永久
swapoff -a && sysctl -w vm.swappiness=0  # 关闭swap
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab  # 取消开机挂载swap

#添加主机名与IP对应关系（记得设置主机名） 如下是案例：
vim /etc/hosts
192.168.20.61 k8s-master
192.168.20.62 k8s-node1
192.168.20.63 k8s-node2

# centos7用户还需要设置路由：
$yum install -y bridge-utils.x86_64

# 加载br_netfilter模块，使用lsmod查看开启的模块:
modprobe br_netfilter

#将桥接的IPv4流量传递到iptables的链：
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

# 重新加载所有配置文件
sysctl --system  
```

 

### 1.1 Docker

```shell
# 卸载已有的docker
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

```shell
#安装依赖
sudo yum update -y && yum install -y yum-utils device-mapper-persistent-data lvm2
#添加阿里云镜像库
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
```

```shell
#安装最新的docker 版本
sudo yum -y install docker-ce docker-ce-cli containerd.io
#查看安装的版本
docker --version
```





##  2 安装Kubeadm，Kubelet和Kubectl

```shell
#master、node节点都需要安装kubelet kubeadm kubectl。
#安装kubernetes的时候，需要安装kubelet, kubeadm等包，但k8s官网给的yum源是#packages.cloud.google.com，国内访问不了，此时我们可以使用阿里云的yum仓库镜像
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```



```txt
kubeadm： 引导集群的命令
kubelet：集群中运行任务的代理程序
kubectl：命令行管理工具
```

```shell
# 安装kubeadm kubelet kubectl 并启动
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
yum install -y kubelet-1.15.0 kubeadm-1.15.0 kubectl-1.15.0   
systemctl enable kubelet && systemctl start kubelet
systemctl enable kubelet.service
```

```shell
#查看安装k8s的安装目录
rpm -ql kubelet
```

### 2.1 Master 服务器

接下来开始下载 k8s 所需要的镜像了，由于都是国外的镜像，导致下载非常麻烦。这里使用国内的镜像，下载完之后改名

```shell
# 列出所需镜像 具体情况每个人可能不一样.列出来的镜像都需要下载
# 大多镜像都是这个域名的k8s.gcr.io 国内下不到
kubeadm config images list 
-------输出------
k8s.gcr.io/kube-apiserver:v1.17.4
k8s.gcr.io/kube-controller-manager:v1.17.4
k8s.gcr.io/kube-scheduler:v1.17.4
k8s.gcr.io/kube-proxy:v1.17.4
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.5
----------------
```

```shell
#使用阿里云的镜像下载 注意版本号 需要对应上面的
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.4
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.4
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.4
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.4
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5
```

```shell
#改名 如果你的和我一样都是下载这些镜像的话 复制粘贴就行
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.4 k8s.gcr.io/kube-apiserver:v1.17.4
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.4 k8s.gcr.io/kube-controller-manager:v1.17.4
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.4 k8s.gcr.io/kube-scheduler:v1.17.4
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.4 k8s.gcr.io/kube-proxy:v1.17.4
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0 k8s.gcr.io/etcd:3.4.3-0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5   k8s.gcr.io/coredns:1.6.5
```

```shell
# 删除原来的镜像, 不然容易出错 一定得删
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.4 
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.4 
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.4 
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.4 
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0 
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5

```

**自此 已完成镜像的下载 。下面是另一种方式 下载镜像**

```shell
#这块是另一种方法 批量 下载 修改 删除
#下载镜像
kubeadm config images list | sed -e 's/^/docker pull /g' -e 's#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/google_containers#g' | sh -x
#修改名称
docker images | grep registry.cn-hangzhou.aliyuncs.com/google_containers | awk '{print "docker tag",$1":"$2,$1":"$2}' | sed -e 's/registry.cn-hangzhou.aliyuncs.com\/google_containers/k8s.gcr.io/2' | sh -x
#删除原镜像
docker images | grep registry.cn-hangzhou.aliyuncs.com/google_containers | awk '{print "docker rmi """$1""":"""$2}' | sh -x
```



### 2.2 Node 服务器

```shell
# 下载
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.4 
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
# 修改镜像tag
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.4 k8s.gcr.io/kube-proxy:v1.17.4 
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
# 删除原来的镜像
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.4 
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1

 docker tag  registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5   k8s.gcr.io/coredns:1.6.5
```



### 2.3 启动

```shell
# 初始化集群  192.168.2.221 是我的 manager 地址
kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-advertise-address=192.168.2.221
# 如果上面初始化失败了可以试试这个 使用的是阿里云镜像
kubeadm init --pod-network-cidr 10.244.0.0/16 --image-repository= registry.cn-hangzhou.aliyuncs.com/google_containers --apiserver-advertise-address=192.168.2.221

# 启动之后会输出很多内容 ,找到类似下面这段命令 ,node 节点加入集群需要运行这段
#kubeadm join 192.168.2.221:6443 --token u1xwqw.lmjnrg45iajz4dvo \
#    --discovery-token-ca-cert-hash #sha256:1b316fc79300c84822d8078116b9edbcb04f2eb25c0c5c552ad6adbd0d8f6f0e 

# 使用kubectl工具

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
```



### 2.3 安装 Pod 网络插件

```shell
# 这一步大概率会因为镜像下载不到问题而导致 节点 notready
# 也可能下载很慢 `quay.io/coreos/flannel:v0.12.0-amd64` 它会反复重试
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```



### 2.4 加入 Node 服务

```shell
# 使用上面 init 命令 之后返回给你的 命令
kubeadm join 192.168.2.221:6443 --token huuvhm.rppy3wyhsfya3d4q \
    --discovery-token-ca-cert-hash sha256:d1c8525bd39f36aad67fdc1cc59101be1a9377cc37c50ba5e25703854d659ea8
```



### 问题解决

```shell
# 如果有问题可能每个人的具体情况不一样, 你需要解决的就是所有 pods 必须是 ready 状态
#查看所有pods
kubectl get pods --all-namespaces
#如果列表中有没有准备好的pods 就会导致 节点 notready
#查看 pods 的日志 看看是什么问题 ,大概率也是镜像下载不到
kubectl describe pods -nkube-system pods-name
```



## 3 部署 Kubernetes dashboard

```shell
#下载文件 如失败 则多重试几次就会成功
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml

```

```yaml
#编辑 vim recommended.yaml 推荐导出到电脑本地来编辑 我这里会显示行号 
# 仅修改 有注释的地方
 39 spec:
 40   type: NodePort # 设置为NodePort 大概这个位置 如果你的没有则手动输入 注意缩进
 41   ports:
 42     - port: 443
 43       nodePort: 30001           # 不设置则为随机，设置则为该端口
 44       targetPort: 8443

# 只能firefox浏览器才能打开，通过修改证书的方式，使得所有浏览器都能打开：注释此段
 50 #apiVersion: v1
 51 #kind: Secret
 52 #metadata:
 53 #  labels:
 54 #    k8s-app: kubernetes-dashboard
 55 #  name: kubernetes-dashboard-certs  #生成证书会用到该名字
 56 #  namespace: kubernetes-dashboard  #生成证书使用该命名空间
 57 #type: Opaque
 
#修改获取镜像相关
193       containers:
194         - name: kubernetes-dashboard
195           image: kubernetesui/dashboard:v2.0.0-beta4
196           #imagePullPolicy: Always       # 注释原来的镜像拉取规则
197           imagePullPolicy: IfNotPresent   # 设置为本地没有则拉取 
```

**手动下载镜像**

```shell
docker pull kubernetesui/dashboard:v2.0.0-beta4
```

**配置证书相关的东西**

```shell
#进入默认证书目录进行配置
cd /etc/kubernetes/pki/
# 查看是否存在namespace为kubernetes-dashboard
kubectl get namespaces
# 不存在namespace为创建kubernetes-dashboard创建namespace
kubectl create namespace kubernetes-dashboard
#创建一个证书
(umask 077; openssl genrsa -out dashboard.key 2048)
#建立证书的签署请求(可以是IP信息等)下面写两个例子
openssl req -new -key dashboard.key -out dashboard.csr -subj "/O=HTI/CN=kubernetes-dashboard"
或我的master节点IP
openssl req -days 3650 -new -out dashboard.csr -key dashboard.key -subj '/CN=192.168.2.221'
#使用集群的ca来签署证书
openssl x509 -req -in dashboard.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dashboard.crt -days 3650
#我们需要把我们创建的证书创建为secret给k8s使用
kubectl create secret generic kubernetes-dashboard-certs --from-file=dashboard.crt=./dashboard.crt --from-file=dashboard.key=./dashboard.key  -n kubernetes-dashboard
在master节点上这是label，拉取镜像
# 设置node选择器label为master
kubectl label node k8s-master type=master
```

**运行 查看**

```shell
kubectl apply -f recommended.yaml
# 所有显示的都必须是ready状态
kubectl get pods -A -o wide
kubectl get svc -n kubernetes-dashboard
```

**创建管理员账号 获取 token 在网页上登录**

```shell
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
# 绑定集群管理员
kubectl create clusterrolebinding dashboard-cluster-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin
#获取token  记录改命令 以后可能会用
kubectl describe secrets  $(kubectl  get secrets -n kubernetes-dashboard | awk  '/dashboard-admin-token/{print $1}' ) -n kubernetes-dashboard |sed -n '/token:.*/p'
```

自此 教程结束

## 4 部署 Redis 主从

**一主两从**

```shell
#主服务器
docker pull docker.io/kubeguide/redis-master:latest
#两个从服务器
docker pull docker.io/kubeguide/guestbook-redis-slave:latest
```

**以下操作均在 主服务器**

找个地方创建 `redis-master-deployment.yaml` 和 `redis-master-service.yaml`

```yaml
#`redis-master-service.yaml`
apiVersion: v1
kind: Service
metadata:
  name: redis-master
  labels:
    app: redis
    role: master
    tier: backend
spec: 
  type: nodePort #指定外部访问类型
  ports: 
  - port: 6379 
    targetPort: 6379 
    NodePort: 30010 # 指定端口号  外部访问 30010 -> 内部6379
  selector: 
    app: redis
    role: master
    tier: backend
```

```yaml
#`redis-master-deployment.yaml`
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: redis-master
spec: 
  replicas: 1
  template: 
    metadata:
      labels:
        app: redis
        role: master
        tier: backend
    spec:
      containers:
      - name: master
        image: docker.io/kubeguide/redis-master:latest
        imagePullPolicy:  
        resources:
          requests:
            cpu: 100m
            memory: 100M
        ports:
        - containerPort: 6379

```

再找个地方 创建 `redis-slave-deployment.yaml` 和 `redis-slave-service.yaml`

```yaml
#redis-slave-deployment.yaml`
apiVersion: extensions/v1beta1 
kind: Deployment
metadata:
  name: redis-slave
spec:
  replicas: 2
  template:
    metadata: 
      labels:
        app: redis
        role: slave
        tier: backend
    spec: 
      containers: 
      - name: slave 
        image: docker.io/kubeguide/guestbook-redis-slave:latest 
        imagePullPolicy: IfNotPresent 
        resources: 
          requests: 
            cpu: 100m
            memory: 100M
        env:
        - name: GET_HOSTS_FROM
          value: env
        ports:
        - containerPort: 6379

```

```yaml
# `redis-slave-service.yaml`
apiVersion: v1
kind: Service 
metadata: 
  name: redis-slave 
  labels: 
    app: redis
    role: slave
    tier: backend
spec: 
  ports: 
  - port: 6379 
  selector:
    app: redis
    role: slave
    tier: backend

```

**直接运行就好了**

```shell
kubectl create -f redis-master-deployment.yaml
kubectl create -f redis-master-service.yaml
kubectl create -f redis-slave-deployment.yaml
kubectl create -f redis-slave-service.yaml
```

**验证**

```shell
# 查看是否有 redis
kubectl get pod
# 进入redis 容器
kubectl exec -it 名称 -- /bin/bash
# 连接控制端
 redis-cli
# set / get 尝试
# 再进入 slave get / set 尝试
```



## 5 部署 Mysql 主从

参考: https://www.jianshu.com/p/acb08a02e4b1

**准备镜像**

```shell
# vi Dockerfile 
FROM mysql:latest
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo "Asia/Shanghai" >> /etc/timezone
```

```shell
#构建
docker build -t xxx/mysql:latest .
```

**准备 mysql master 的 deployment 和 service 文件**

mysql-master-deployment.yaml

```yaml
apiVersion: extensions/v1beta1 # 如果你的是1.8以上的 k8s 请使用 apps/v1
#apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: mysql
  template:
    metadata:
      labels:
        tier: mysql
    spec:
      containers:
      - name: mysql-container
        image: xxx/mysql:latest # 刚刚构建的镜像
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456" # 登录密码
        - name: lower_case_table_names
          value: "1"
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: mysqldb
        - mountPath: /etc/mysql/mysql.conf.d
          name: mysqlcfg
      volumes:
      - name: mysqldb
        hostPath:
          path: /usr/local/mysql/database/data
      - name: mysqlcfg
        hostPath:
          path: /usr/local/mysql/database/conf
```

mysql-master-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-master
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 3306 
    targetPort: 3306
    name: mysql-port
    nodePort: 30020 # 暴露端口 
  selector:
    tier: mysql
```

创建刚刚挂在的目录 , 主从服务器都要创建  

**创建 mysqld.cnf**

```shell
# 在/usr/local/mysql/database/conf 下创建 
[mysql]
default-character-set = utf8

[mysql.server]
default-character-set = utf8

[mysqld_safe]
default-character-set = utf8

[client]
default-character-set = utf8

[mysqld]
log-bin=mysql-bin # 开启binlog
binlog-ignore-db=mysql
server-id=1001
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
character_set_server = utf8
lower_case_table_names=1
symbolic-links=0
sql_mode = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

**准备 mysql-slave-deployment 和 mysql-slave-service**

mysql-slave-deployment

```yaml
apiVersion: extensions/v1beta1 # 这里注意版本引起的问题
kind: Deployment
metadata:
  name: mysql-slave-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: mysql-slave
  template:
    metadata:
      labels:
        tier: mysql-slave
    spec:
      containers:
      - name: mysql-slave-container
        image: xxx/mysql:latest # 镜像 你生成的是啥就写啥
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        - name: lower_case_table_names
          value: "1"  
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: mysqldb
        - mountPath: /etc/mysql/mysql.conf.d
          name: mysqlcfg
      volumes:
      - name: mysqldb
        hostPath:
          path: /usr/local/mysql/database/data
      - name: mysqlcfg
        hostPath:
          path: /usr/local/mysql/database/conf
```

mysql-slave-service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-slave-service
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 3307 
    targetPort: 3306
    name: mysql-slave-port
    nodePort: 3307
  selector:
    tier: mysql-slave
```



**全部运行**

```shell
kubectl create -f mysql-deployment.yaml
kubectl create -f mysql-service.yaml
kubectl create -f mysql-slave-deployment.yaml 
kubectl create -f mysql-slave-service.yaml
```



**进入 master 数据库 配置 只读账号 ,查看 binlog**

```shell
# 查看 mysql 请确保无误运行
kubectl get pod | grep mysql

# 进入 master 容器
kubectl exec -it mysql-deployment-7cbc7f69f-hf6rq -- mysql -u root -p

# 配置只读账号
CREATE USER 'reader'@'%' IDENTIFIED WITH mysql_native_password BY '123456'
GRANT REPLICATION SLAVE ON *.* TO 'reader'@'%';
FLUSH PRIVILEGES;

# 查看binlog
show master status
#记住 file 列 和 position 的内容
```

**进入从数据库 配置 只读账号 , 连接 master 数据库, 开始 slave**

```shell
# 配置账号
CREATE USER 'reader'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
GRANT SELECT, REPLICATION SLAVE ON *.* TO `reader`@`%` ;
FLUSH PRIVILEGES;

# 连接 master 数据库  内容根据实际情况修改
change master to master_host='192.168.2.223',
master_user='reader',
master_password='123456',
master_port=30020,
master_log_file='binlog.000002',master_log_pos=7004;

# 开始
start slave;

# 查看执行情况 根据实际情况解决问题
show slave status\G;

# 判断成功与否条件
# Slave_IO_Running: Yes
# Slave_SQL_Running: Yes
# 8.0 的 mysql 不要修改root账号的密码模式,slave 扩展的时候启动会吧 root 账号忽略导致无法登陆
```

