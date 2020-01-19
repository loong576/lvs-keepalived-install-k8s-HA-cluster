# 一、部署环境

## 1.1 主机列表

|      主机名      | Centos版本 |      ip       | docker version | flannel version | Keepalived version | 主机配置 |              备注              |
| :--------------: | :--------: | :-----------: | :------------: | :-------------: | :----------------: | :------: | :----------------------------: |
| lvs-keepalived01 |  7.6.1810  | 172.27.34.28  |       /        |        /        |       v1.3.5       |   4C4G   |         lvs-keepalived         |
| lvs-keepalived01 |  7.6.1810  | 172.27.34.29  |       /        |        /        |       v1.3.5       |   4C4G   |         lvs-keepalived         |
|     master01     |  7.6.1810  | 172.27.34.35  |    18.09.9     |     v0.11.0     |                    |   4C4G   |         control plane          |
|     master02     |  7.6.1810  | 172.27.34.36  |    18.09.9     |     v0.11.0     |         /          |   4C4G   |         control plane          |
|     master03     |  7.6.1810  | 172.27.34.37  |    18.09.9     |     v0.11.0     |         /          |   4C4G   |         control plane          |
|      work01      |  7.6.1810  | 172.27.34.161 |    18.09.9     |        /        |         /          |   4C4G   |          worker nodes          |
|      work02      |  7.6.1810  | 172.27.34.162 |    18.09.9     |        /        |         /          |   4C4G   |          worker nodes          |
|      work03      |  7.6.1810  | 172.27.34.163 |    18.09.9     |        /        |         /          |   4C4G   |          worker nodes          |
|       VIP        |  7.6.1810  | 172.27.34.222 |       /        |        /        |       v1.3.5       |   4C4G   | 在lvs-keepalived两台主机上浮动 |
|      client      |  7.6.1810  | 172.27.34.85  |       /        |        /        |         /          |   4C4G   |             client             |

共有9台服务器，2台为lvs-keepalived集群，3台control plane集群，3台work集群，1台client。

## 1.2 k8s 版本

|  主机名  | kubelet version | kubeadm version | kubectl version | 备注        |
| :------: | :-------------: | :-------------: | :-------------: | ----------- |
| master01 |     v1.16.4     |     v1.16.4     |     v1.16.4     | kubectl选装 |
| master02 |     v1.16.4     |     v1.16.4     |     v1.16.4     | kubectl选装 |
| master03 |     v1.16.4     |     v1.16.4     |     v1.16.4     | kubectl选装 |
|  work01  |     v1.16.4     |     v1.16.4     |     v1.16.4     | kubectl选装 |
|  work02  |     v1.16.4     |     v1.16.4     |     v1.16.4     | kubectl选装 |
|  work03  |     v1.16.4     |     v1.16.4     |     v1.16.4     | kubectl选装 |
|  client  |        /        |        /        |     v1.16.4     | client      |



# 二、高可用架构

## 1. 架构图

本文采用kubeadm方式搭建高可用k8s集群，k8s集群的高可用实际是k8s各核心组件的高可用，这里使用**集群**模式(针对apiserver来讲)，架构如下：

![图片.png](https://ask.qcloudimg.com/draft/6211241/17fkined3k.png)

## 2. 集群模式高可用架构说明

|      核心组件      | 高可用模式 | 高可用实现方式  |
| :----------------: | :--------: | :-------------: |
|     apiserver      |    集群    | lvs+keepalived  |
| controller-manager |    主备    | leader election |
|     scheduler      |    主备    | leader election |
|        etcd        |    集群    |     kubeadm     |




> - **apiserver**  通过lvs-keepalived实现高可用，vip将请求分发至各个control plane节点的apiserver组件；
> - **controller-manager**  k8s内部通过选举方式产生领导者(由--leader-elect 选型控制，默认为true)，同一时刻集群内只有一个controller-manager组件运行；
> - **scheduler**  k8s内部通过选举方式产生领导者(由--leader-elect 选型控制，默认为true)，同一时刻集群内只有一个scheduler组件运行；
> - **etcd**  通过运行kubeadm方式自动创建集群来实现高可用，部署的节点数为奇数，3节点方式最多容忍一台机器宕机。

# 三、Centos7.6安装

本文所有的服务器都为Centos7.6，**Centos7.6安装详见：**[Centos7.6操作系统安装及优化全纪录 ](https://blog.51cto.com/3241766/2398136)

安装Centos时已经禁用了防火墙和selinux并设置了阿里源。

# 四、k8s集群安装准备工作

**control plane和work节点都执行本部分操作，以master01为例记录搭建过程。**

## 1. 配置主机名

### 1.1 修改主机名

```bash
[root@centos7 ~]# hostnamectl set-hostname master01
[root@centos7 ~]# more /etc/hostname             
master01
```

退出重新登陆即可显示新设置的主机名master01，各服务器修改为对应的主机名。

### 1.2 修改hosts文件

```bash
[root@master01 ~]# cat >> /etc/hosts << EOF
172.27.34.35    master01
172.27.34.36   master02
172.27.34.37    master03
172.27.34.161   work01 
172.27.34.162   work02
172.27.34.163   work03
EOF
```

![图片.png](https://ask.qcloudimg.com/draft/6211241/tq0leua6mj.png)

## 2. 验证mac地址uuid

```bash
[root@master01 ~]# cat /sys/class/net/ens160/address
[root@master01 ~]# cat /sys/class/dmi/id/product_uuid
```

![图片.png](https://ask.qcloudimg.com/draft/6211241/a8lrud0fim.png)

保证各节点mac和uuid唯一

## 3. 禁用swap

### 3.1 临时禁用

```bash
[root@master01 ~]# swapoff -a
```

### 3.2 永久禁用

若需要重启后也生效，在禁用swap后还需修改配置文件/etc/fstab，注释swap

```bash
[root@master01 ~]# sed -i.bak '/swap/s/^/#/' /etc/fstab
```

![图片.png](https://ask.qcloudimg.com/draft/6211241/gpcqscpco2.png)

## 4. 内核参数修改

本文的k8s网络使用flannel，该网络需要设置内核参数bridge-nf-call-iptables=1，修改这个参数需要系统有br_netfilter模块。

### 4.1 br_netfilter模块加载

**查看br_netfilter模块：**

```bash
[root@master01 ~]# lsmod |grep br_netfilter
```

如果系统没有br_netfilter模块则执行下面的新增命令，如有则忽略。

**临时新增br_netfilter模块:**

```bash
[root@master01 ~]# modprobe br_netfilter
```

该方式重启后会失效

**永久新增br_netfilter模块：**

```bash
[root@master01 ~]# cat > /etc/rc.sysinit << EOF
#!/bin/bash
for file in /etc/sysconfig/modules/*.modules ; do
[ -x $file ] && $file
done
EOF
[root@master01 ~]# cat > /etc/sysconfig/modules/br_netfilter.modules << EOF
modprobe br_netfilter
EOF
[root@master01 ~]# chmod 755 /etc/sysconfig/modules/br_netfilter.modules
```

![图片.png](https://ask.qcloudimg.com/draft/6211241/sq78j6wvdc.png)

### 4.2 内核参数临时修改

```bash
[root@master01 ~]# sysctl net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-iptables = 1
[root@master01 ~]# sysctl net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-ip6tables = 1

```

### 4.3 内核参数永久修改

```bash
[root@master01 ~]# cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
[root@master01 ~]# sysctl -p /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/thudn6to7d.png)

## 5. 设置kubernetes源

### 5.1 新增kubernetes源

```bash
[root@master01 ~]# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

```

> - [] 中括号中的是repository id，唯一，用来标识不同仓库
> - name  仓库名称，自定义
> - baseurl 仓库地址
> - enable 是否启用该仓库，默认为1表示启用
> - gpgcheck 是否验证从该仓库获得程序包的合法性，1为验证
> - repo_gpgcheck 是否验证元数据的合法性 元数据就是程序包列表，1为验证
> - gpgkey=URL 数字签名的公钥文件所在位置，如果gpgcheck值为1，此处就需要指定gpgkey文件的位置，如果gpgcheck值为0就不需要此项了

### 5.2 更新缓存

```bash
[root@master01 ~]# yum clean all
[root@master01 ~]# yum -y makecache
```

## 6. 免密登录

配置master01到master02、master03免密登录，本步骤只在master01上执行。

### 6.1 创建秘钥

```bash
[root@master01 ~]# ssh-keygen -t rsa
```

![图片.png](https://ask.qcloudimg.com/draft/6211241/30fme7fera.png)

### 6.2 将秘钥同步至master02/master03

```bash
[root@master01 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@172.27.34.35
[root@master01 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@172.27.34.36

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/pgbd0n9r5j.png)

![图片.png](https://ask.qcloudimg.com/draft/6211241/051koqudrd.png)

### 6.3 免密登陆测试

```bash
[root@master01 ~]# ssh 172.27.34.36
[root@master01 ~]# ssh master03

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/zxp4cnampa.png)

master01可以直接登录master02和master03，不需要输入密码。

## 7. 服务器重启

重启各control plane和work节点。

# 五、Docker安装

**control plane和work节点都执行本部分操作。**

## 1. 安装依赖包

```bash
[root@master01 ~]# yum install -y yum-utils   device-mapper-persistent-data   lvm2

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/lzz9kvwu31.png)

## 2. 设置Docker源

```bash
[root@master01 ~]# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/sqx0do0owf.png)

## 3. 安装Docker CE

### 3.1 docker安装版本查看

```bash
[root@master01 ~]# yum list docker-ce --showduplicates | sort -r

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/g1zf4wzw8p.png)

### 3.2 安装docker

```bash
[root@master01 ~]# yum install docker-ce-18.09.9 docker-ce-cli-18.09.9 containerd.io -y

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/6t1rwssd84.png)
指定安装的docker版本为18.09.9

## 4. 启动Docker

```bash
[root@master01 ~]# systemctl start docker
[root@master01 ~]# systemctl enable docker

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/gb2m14dji4.png)

## 5. 命令补全

### 5.1 安装bash-completion

```bash
[root@master01 ~]# yum -y install bash-completion

```

### 5.2 加载bash-completion

```bash
[root@master01 ~]# source /etc/profile.d/bash_completion.sh

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/pssxjfra0j.png)

## 6. 镜像加速

由于Docker Hub的服务器在国外，下载镜像会比较慢，可以配置镜像加速器。主要的加速器有：Docker官方提供的中国registry mirror、阿里云加速器、DaoCloud 加速器，本文以阿里加速器配置为例。

### 6.1 登陆阿里云容器模块

登陆地址为：https://cr.console.aliyun.com ,未注册的可以先注册阿里云账户

![图片.png](https://ask.qcloudimg.com/draft/6211241/zfpp5kt87u.png)

### 6.2 配置镜像加速器

**配置daemon.json文件**

```bash
[root@master01 ~]# mkdir -p /etc/docker
[root@master01 ~]# tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://v16stybc.mirror.aliyuncs.com"]
}
EOF

```

**重启服务**

```bash
[root@master01 ~]# systemctl daemon-reload
[root@master01 ~]# systemctl restart docker

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/an0wlxt82l.png)

加速器配置完成

## 7. 验证

```bash
[root@master01 ~]# docker --version
[root@master01 ~]# docker run hello-world

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/iqgxiebln0.png)

通过查询docker版本和运行容器hello-world来验证docker是否安装成功。

## 8. 修改Cgroup Driver

### 8.1 修改daemon.json

修改daemon.json，新增‘"exec-opts": ["native.cgroupdriver=systemd"’

```bash
[root@master01 ~]# more /etc/docker/daemon.json 
{
  "registry-mirrors": ["https://v16stybc.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}

```

### 8.2 重新加载docker

```bash
[root@master01 ~]# systemctl daemon-reload
[root@master01 ~]# systemctl restart docker

```

修改cgroupdriver是为了消除告警：
[WARNING IsDockerSystemdCheck]: detected   "cgroupfs" as the Docker cgroup driver. The recommended driver is   "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/

# 六、k8s安装

**control plane和work节点都执行本部分操作**。

## 1. 版本查看

```bash
[root@master01 ~]# yum list kubelet --showduplicates | sort -r

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/5pnhqmbgfl.png)

本文安装的kubelet版本是1.16.4，该版本支持的docker版本为1.13.1, 17.03, 17.06, 17.09, 18.06, 18.09。

## 2. 安装kubelet、kubeadm和kubectl

### 2.1 安装三个包

```bash
[root@master01 ~]# yum install -y kubelet-1.16.4 kubeadm-1.16.4 kubectl-1.16.4

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/ruuut36cdn.png)

### 2.2 安装包说明

> - **kubelet**  运行在集群所有节点上，用于启动Pod和容器等对象的工具
> - **kubeadm**  用于初始化集群，启动集群的命令工具
> - **kubectl**  用于和集群通信的命令行，通过kubectl可以部署和管理应用，查看各种资源，创建、删除和更新各种组件

### 2.3 启动kubelet

启动kubelet并设置开机启动

```bash
[root@master01 ~]# systemctl enable kubelet && systemctl start kubelet

```

### 2.4 kubectl命令补全

```bash
[root@master01 ~]# echo "source <(kubectl completion bash)" >> ~/.bash_profile
[root@master01 ~]# source .bash_profile 

```

## 3. 下载镜像

### 3.1 镜像下载的脚本

Kubernetes几乎所有的安装组件和Docker镜像都放在goolge自己的网站上,直接访问可能会有网络问题，这里的解决办法是从阿里云镜像仓库下载镜像，拉取到本地以后改回默认的镜像tag。本文通过运行image.sh脚本方式拉取镜像。

```bash
[root@master01 ~]# more image.sh 
#!/bin/bash
url=registry.cn-hangzhou.aliyuncs.com/loong576
version=v1.16.4
images=(`kubeadm config images list --kubernetes-version=$version|awk -F '/' '{print $2}'`)
for imagename in ${images[@]} ; do
  docker pull $url/$imagename
  docker tag $url/$imagename k8s.gcr.io/$imagename
  docker rmi -f $url/$imagename
done

```

url为阿里云镜像仓库地址，version为安装的kubernetes版本。

### 3.2 下载镜像

运行脚本image.sh，下载指定版本的镜像

```bash
[root@master01 ~]# ./image.sh
[root@master01 ~]# docker images

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/v1dyvwn5qq.png)

# 七、初始化Master

**master01节点执行本部分操作。**

## 1. kubeadm.conf

```bash
[root@master01 ~]# more kubeadm-config.yaml 
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.16.4
apiServer:
  certSANs:    #填写所有kube-apiserver节点的hostname、IP、VIP
  - master01
  - master02
  - master03
  - work01
  - work02
  - work03
  - 172.27.34.35
  - 172.27.34.36
  - 172.27.34.37
  - 172.27.34.161
  - 172.27.34.162
  - 172.27.34.163
  - 172.27.34.222
controlPlaneEndpoint: "172.27.34.222:6443"
networking:
  podSubnet: "10.244.0.0/16"
```

![图片.png](https://ask.qcloudimg.com/draft/6211241/0xwsw9b9x3.png)

kubeadm.conf为初始化的配置文件

## 2. master01起虚ip

在master01上起虚ip：172.27.34.222

```bash
[root@master01 ~]# ifconfig ens160:2 172.27.34.222 netmask 255.255.255.0 up

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/6zym4457wh.png)

起虚ip目的是为了执行master01的初始化，待初始化完成后去掉该虚ip

## 3. master初始化

```bash
[root@master01 ~]# kubeadm init --config=kubeadm-config.yaml

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/xdgvkj4r72.png)

记录kubeadm join的输出，后面需要这个命令将work节点和其他control plane节点加入集群中。

```bash
You can now join any number of control-plane nodes by copying certificate authorities 
and service account keys on each node and then running the following as root:

  kubeadm join 172.27.34.222:6443 --token lw90fv.j1lease5jhzj9ih2 \
    --discovery-token-ca-cert-hash sha256:79575e7a39eac086e121364f79e58a33f9c9de2a4e9162ad81d0abd1958b24f4 \
    --control-plane       

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.27.34.222:6443 --token lw90fv.j1lease5jhzj9ih2 \
    --discovery-token-ca-cert-hash sha256:79575e7a39eac086e121364f79e58a33f9c9de2a4e9162ad81d0abd1958b24f4 

```

**初始化失败：**

如果初始化失败，可执行kubeadm reset后重新初始化

```
[root@master01 ~]# kubeadm reset
[root@master01 ~]# rm -rf $HOME/.kube/config

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/y928iw132n.png)

## 4. 加载环境变量

```bash
[root@master01 ~]# echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
[root@master01 ~]# source .bash_profile

```

本文所有操作都在root用户下执行，若为非root用户，则执行如下操作：

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

```

## 5. 安装flannel网络

在master01上新建flannel网络

```bash
[root@master01 ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/smrb9feqer.png)

由于网络原因，可能会安装失败，可以在文末直接下载kube-flannel.yml文件，然后再执行apply

# 八、control plane节点加入k8s集群

## 1. 证书分发

### 1.1 master01分发证书

在master01上运行脚本cert-main-master.sh，将证书分发至master02和master03

```bash
[root@master01 ~]# ll|grep cert-main-master.sh 
-rwxr--r--  1 root root     638 1月  16 10:25 cert-main-master.sh
[root@master01 ~]# more cert-main-master.sh
USER=root # customizable
CONTROL_PLANE_IPS="172.27.34.36 172.27.34.37"
for host in ${CONTROL_PLANE_IPS}; do
    scp /etc/kubernetes/pki/ca.crt "${USER}"@$host:
    scp /etc/kubernetes/pki/ca.key "${USER}"@$host:
    scp /etc/kubernetes/pki/sa.key "${USER}"@$host:
    scp /etc/kubernetes/pki/sa.pub "${USER}"@$host:
    scp /etc/kubernetes/pki/front-proxy-ca.crt "${USER}"@$host:
    scp /etc/kubernetes/pki/front-proxy-ca.key "${USER}"@$host:
    scp /etc/kubernetes/pki/etcd/ca.crt "${USER}"@$host:etcd-ca.crt
    # Quote this line if you are using external etcd
    scp /etc/kubernetes/pki/etcd/ca.key "${USER}"@$host:etcd-ca.key
done

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/1252ixgb5j.png)

### 1.2 master02移动证书至指定目录

在master02上运行脚本cert-other-master.sh，将证书移至指定目录

```bash
[root@master02 ~]# more cert-other-master.sh 
USER=root # customizable
mkdir -p /etc/kubernetes/pki/etcd
mv /${USER}/ca.crt /etc/kubernetes/pki/
mv /${USER}/ca.key /etc/kubernetes/pki/
mv /${USER}/sa.pub /etc/kubernetes/pki/
mv /${USER}/sa.key /etc/kubernetes/pki/
mv /${USER}/front-proxy-ca.crt /etc/kubernetes/pki/
mv /${USER}/front-proxy-ca.key /etc/kubernetes/pki/
mv /${USER}/etcd-ca.crt /etc/kubernetes/pki/etcd/ca.crt
# Quote this line if you are using external etcd
mv /${USER}/etcd-ca.key /etc/kubernetes/pki/etcd/ca.key
[root@master02 ~]# ./cert-other-master.sh 

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/tcakvfjn04.png)

### 1.3 master03移动证书至指定目录

在master03上也运行脚本cert-other-master.sh

```bash
[root@master03 ~]# pwd
/root
[root@master03 ~]# ll|grep cert-other-master.sh 
-rwxr--r--  1 root root  484 1月  16 10:30 cert-other-master.sh
[root@master03 ~]# ./cert-other-master.sh 

```

## 2. master02加入k8s集群

```bash
[root@master03 ~]# kubeadm join 172.27.34.222:6443 --token lw90fv.j1lease5jhzj9ih2     --discovery-token-ca-cert-hash sha256:79575e7a39eac086e121364f79e58a33f9c9de2a4e9162ad81d0abd1958b24f4     --control-plane

```

运行初始化master生成的control plane节点加入集群的命令

![图片.png](https://ask.qcloudimg.com/draft/6211241/a29ucs429e.png)

## 3. master03加入k8s集群

```bash
[root@master03 ~]# kubeadm join 172.27.34.222:6443 --token 0p7rzn.fdanprq4y8na36jh     --discovery-token-ca-cert-hash sha256:fc7a828208d554329645044633159e9dc46b0597daf66769988fee8f3fc0636b     --control-plane

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/zdqlt5hwhi.png)

## 4. 加载环境变量

master02和master03加载环境变量

```bash
[root@master02 ~]# scp master01:/etc/kubernetes/admin.conf /etc/kubernetes/
[root@master02 ~]# echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
[root@master02 ~]# source .bash_profile 
[root@master03 ~]# scp master01:/etc/kubernetes/admin.conf /etc/kubernetes/
[root@master03 ~]# echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
[root@master03 ~]# source .bash_profile 

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/j44mbth4bm.png)

该步操作是为了在master02和master03上也能执行kubectl命令。

## 5. k8s集群节点查看

```bash
[root@master01 ~]# kubectl get nodes
[root@master01 ~]# kubectl get po -o wide -n kube-system 

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/svlvghzp3i.png)发现master01和master03下载flannel异常，分别在master01和master03上手动下载该镜像后正常。

```bash
[root@master01 ~]# docker pull  registry.cn-hangzhou.aliyuncs.com/loong576/flannel:v0.11.0-amd64
[root@master03 ~]# docker pull  registry.cn-hangzhou.aliyuncs.com/loong576/flannel:v0.11.0-amd64

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/gaxby6m7wg.png)

# 九、work节点加入k8s集群

## 1. work01加入k8s集群

```bash
[root@work01 ~]# kubeadm join 172.27.34.222:6443 --token lw90fv.j1lease5jhzj9ih2     --discovery-token-ca-cert-hash sha256:79575e7a39eac086e121364f79e58a33f9c9de2a4e9162ad81d0abd1958b24f4

```

运行初始化master生成的work节点加入集群的命令

![图片.png](https://ask.qcloudimg.com/draft/6211241/rpp86bmtjc.png)

## 2. work02加入k8s集群

```bash
[root@work02 ~]# kubeadm join 172.27.34.222:6443 --token lw90fv.j1lease5jhzj9ih2     --discovery-token-ca-cert-hash sha256:79575e7a39eac086e121364f79e58a33f9c9de2a4e9162ad81d0abd1958b24f4

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/hb30i776j0.png)

## 3. work03加入k8s集群

```bash
[root@work03 ~]# kubeadm join 172.27.34.222:6443 --token lw90fv.j1lease5jhzj9ih2     --discovery-token-ca-cert-hash sha256:79575e7a39eac086e121364f79e58a33f9c9de2a4e9162ad81d0abd1958b24f4

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/g789177lu3.png)

## 4. k8s集群各节点查看

```bash
[root@master01 ~]# kubectl get nodes
[root@master01 ~]# kubectl get po -o wide -n kube-system 

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/kk07xnd73e.png)



# 十、ipvs安装

**lvs-keepalived01和lvs-keepalived02都执行本操作。**

## 1. 安装ipvs

LVS无需安装，安装的是管理工具，第一种叫ipvsadm，第二种叫keepalive。ipvsadm是通过命令行管理，而keepalive读取配置文件管理。

```bash
[root@lvs-keepalived01 ~]# yum -y install ipvsadm

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/ebjwpi2fp7.png)

## 2. 加载ipvsadm模块

把ipvsadm模块加载进系统

```bash
[root@lvs-keepalived01 ~]# ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
[root@lvs-keepalived01 ~]# lsmod | grep ip_vs
ip_vs                 145497  0 
nf_conntrack          133095  1 ip_vs
libcrc32c              12644  3 xfs,ip_vs,nf_conntrack

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/eh8yp6eq85.png)

**lvs相关实践详见：**[LVS+Keepalived+Nginx负载均衡搭建测试](https://blog.51cto.com/3241766/2094750)

# 十一、keepalived安装

**lvs-keepalived01和lvs-keepalived02都执行本操作。**

## 1. keepalived安装

```bash
[root@lvs-keepalived01 ~]# yum -y install keepalived

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/489rqyte47.png)

## 2. keepalived配置

lvs-keepalived01配置如下：

```bash
[root@lvs-keepalived01 ~]# more /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
   router_id lvs-keepalived01   #router_id 机器标识，通常为hostname，但不一定非得是hostname。故障发生时，邮件通知会用到。
}
vrrp_instance VI_1 {            #vrrp实例定义部分
    state MASTER                #设置lvs的状态，MASTER和BACKUP两种，必须大写 
    interface ens160            #设置对外服务的接口
    virtual_router_id 100       #设置虚拟路由标示，这个标示是一个数字，同一个vrrp实例使用唯一标示 
    priority 100                #定义优先级，数字越大优先级越高，在一个vrrp——instance下，master的优先级必须大于backup
    advert_int 1                #设定master与backup负载均衡器之间同步检查的时间间隔，单位是秒
    authentication {            #设置验证类型和密码
        auth_type PASS          #主要有PASS和AH两种
        auth_pass 1111          #验证密码，同一个vrrp_instance下MASTER和BACKUP密码必须相同
    }
    virtual_ipaddress {         #设置虚拟ip地址，可以设置多个，每行一个
        172.27.34.222
    }
}
virtual_server 172.27.34.222 6443 {  #设置虚拟服务器，需要指定虚拟ip和服务端口
    delay_loop 6                     #健康检查时间间隔
    lb_algo wrr                      #负载均衡调度算法
    lb_kind DR                       #负载均衡转发规则
    #persistence_timeout 50          #设置会话保持时间，对动态网页非常有用
    protocol TCP                     #指定转发协议类型，有TCP和UDP两种
    real_server 172.27.34.35 6443 {  #配置服务器节点1，需要指定real server的真实IP地址和端口
    weight 10                        #设置权重，数字越大权重越高
    TCP_CHECK {                      #realserver的状态监测设置部分单位秒
       connect_timeout 10            #连接超时为10秒
       retry 3                       #重连次数
       delay_before_retry 3          #重试间隔
       connect_port 6443             #连接端口为6443，要和上面的保持一致
       }
    }
    real_server 172.27.34.36 6443 {  #配置服务器节点1，需要指定real server的真实IP地址和端口
    weight 10                        #设置权重，数字越大权重越高
    TCP_CHECK {                      #realserver的状态监测设置部分单位秒
       connect_timeout 10            #连接超时为10秒
       retry 3                       #重连次数
       delay_before_retry 3          #重试间隔
       connect_port 6443             #连接端口为6443，要和上面的保持一致
       }
    }
    real_server 172.27.34.37 6443 {  #配置服务器节点1，需要指定real server的真实IP地址和端口
    weight 10                        #设置权重，数字越大权重越高
    TCP_CHECK {                      #realserver的状态监测设置部分单位秒
       connect_timeout 10            #连接超时为10秒
       retry 3                       #重连次数
       delay_before_retry 3          #重试间隔
       connect_port 6443             #连接端口为6443，要和上面的保持一致
       }
    }
}

```

lvs-keepalived02配置如下：

```bash
[root@lvs-keepalived02 ~]# more /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
   router_id lvs-keepalived02   #router_id 机器标识，通常为hostname，但不一定非得是hostname。故障发生时，邮件通知会用到。
}
vrrp_instance VI_1 {            #vrrp实例定义部分
    state BACKUP                #设置lvs的状态，MASTER和BACKUP两种，必须大写 
    interface ens160            #设置对外服务的接口
    virtual_router_id 100       #设置虚拟路由标示，这个标示是一个数字，同一个vrrp实例使用唯一标示 
    priority 90                 #定义优先级，数字越大优先级越高，在一个vrrp——instance下，master的优先级必须大于backup
    advert_int 1                #设定master与backup负载均衡器之间同步检查的时间间隔，单位是秒
    authentication {            #设置验证类型和密码
        auth_type PASS          #主要有PASS和AH两种
        auth_pass 1111          #验证密码，同一个vrrp_instance下MASTER和BACKUP密码必须相同
    }
    virtual_ipaddress {         #设置虚拟ip地址，可以设置多个，每行一个
        172.27.34.222
    }
}
virtual_server 172.27.34.222 6443 {  #设置虚拟服务器，需要指定虚拟ip和服务端口
    delay_loop 6                     #健康检查时间间隔
    lb_algo wrr                      #负载均衡调度算法
    lb_kind DR                       #负载均衡转发规则
    #persistence_timeout 50          #设置会话保持时间，对动态网页非常有用
    protocol TCP                     #指定转发协议类型，有TCP和UDP两种
    real_server 172.27.34.35 6443 {  #配置服务器节点1，需要指定real server的真实IP地址和端口
    weight 10                        #设置权重，数字越大权重越高
    TCP_CHECK {                      #realserver的状态监测设置部分单位秒
       connect_timeout 10            #连接超时为10秒
       retry 3                       #重连次数
       delay_before_retry 3          #重试间隔
       connect_port 6443             #连接端口为6443，要和上面的保持一致
       }
    }
    real_server 172.27.34.36 6443 {  #配置服务器节点1，需要指定real server的真实IP地址和端口
    weight 10                        #设置权重，数字越大权重越高
    TCP_CHECK {                      #realserver的状态监测设置部分单位秒
       connect_timeout 10            #连接超时为10秒
       retry 3                       #重连次数
       delay_before_retry 3          #重试间隔
       connect_port 6443             #连接端口为6443，要和上面的保持一致
       }
    }
    real_server 172.27.34.37 6443 {  #配置服务器节点1，需要指定real server的真实IP地址和端口
    weight 10                        #设置权重，数字越大权重越高
    TCP_CHECK {                      #realserver的状态监测设置部分单位秒
       connect_timeout 10            #连接超时为10秒
       retry 3                       #重连次数
       delay_before_retry 3          #重试间隔
       connect_port 6443             #连接端口为6443，要和上面的保持一致
       }
    }
}

```

## 3. master01上去掉vip

```bash
[root@master01 ~]# ifconfig ens160:2 172.27.34.222 netmask 255.255.255.0 down

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/q09fqto37n.png)

master01上去掉初始化使用的ip 172.27.34.222

## 4. 启动keepalived

lvs-keepalived01和lvs-keepalived02都启动keepalived并设置为开机启动

```bash
[root@lvs-keepalived01 ~]# service keepalived start
Redirecting to /bin/systemctl start keepalived.service
[root@lvs-keepalived01 ~]# systemctl enable keepalived
Created symlink from /etc/systemd/system/multi-user.target.wants/keepalived.service to /usr/lib/systemd/system/keepalived.service.

```

## 5. vip查看

```bash
[root@lvs-keepalived01 ~]# ip a

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/dnyeijsigd.png)

此时vip在lvs-keepalived01上



# 十二、control plane节点配置

**control plane都执行本操作。**

## 1. 新建realserver.sh

打开control plane所在服务器的“路由”功能、关闭“ARP查询”功能并设置回环ip，三台control plane配置相同，如下：

```bash
[root@master01 ~]# cd /etc/rc.d/init.d/
[root@master01 init.d]# more realserver.sh 
#!/bin/bash
    SNS_VIP=172.27.34.222
    case "$1" in
    start)
        ifconfig lo:0 $SNS_VIP netmask 255.255.255.255 broadcast $SNS_VIP
        /sbin/route add -host $SNS_VIP dev lo:0
        echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
        echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
        echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
        echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
        sysctl -p >/dev/null 2>&1
        echo "RealServer Start OK"
        ;;
    stop)
        ifconfig lo:0 down
        route del $SNS_VIP >/dev/null 2>&1
        echo "0" >/proc/sys/net/ipv4/conf/lo/arp_ignore
        echo "0" >/proc/sys/net/ipv4/conf/lo/arp_announce
        echo "0" >/proc/sys/net/ipv4/conf/all/arp_ignore
        echo "0" >/proc/sys/net/ipv4/conf/all/arp_announce
        echo "RealServer Stoped"
        ;;
    *)
        echo "Usage: $0 {start|stop}"
        exit 1
    esac
    exit 0

```

此脚本用于control plane节点绑定 VIP ，并抑制响应 VIP 的 ARP 请求。这样做的目的是为了不让关于 VIP 的 ARP 广播时，节点服务器应答（ 因为control plane节点都绑定了 VIP ，如果不做设置它们会应答，就会乱套 ）。

## 2 运行realserver.sh脚本

在所有control plane节点执行realserver.sh脚本：

```bash
[root@master01 init.d]# chmod u+x realserver.sh 
[root@master01 init.d]# /etc/rc.d/init.d/realserver.sh start
RealServer Start OK

```

给realserver.sh脚本授予执行权限并运行realserver.sh脚本

![图片.png](https://ask.qcloudimg.com/draft/6211241/x43m5gvqwl.png)

## 3. realserver.sh开启启动

```bash
[root@master01 init.d]# sed -i '$a /etc/rc.d/init.d/realserver.sh start' /etc/rc.d/rc.local
[root@master01 init.d]# chmod u+x /etc/rc.d/rc.local 

```

# 十三、client配置

## 1. 设置kubernetes源

### 1.1 新增kubernetes源

```bash
[root@client ~]# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/g5i1vrmc55.png)

### 1.2 更新缓存

```bash
[root@client ~]# yum clean all
[root@client ~]# yum -y makecache

```

## 2. 安装kubectl

```bash
[root@client ~]# yum install -y kubectl-1.16.4

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/yikg3wasmd.png)

安装版本与集群版本保持一致

## 3. 命令补全

### 3.1 安装bash-completion

```bash
[root@client ~]# yum -y install bash-completion

```

### 3.2 加载bash-completion

```bash
[root@client ~]# source /etc/profile.d/bash_completion.sh

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/uoid2czypj.png)

### 3.3 拷贝admin.conf

```bash
[root@client ~]# mkdir -p /etc/kubernetes
[root@client ~]# scp 172.27.34.35:/etc/kubernetes/admin.conf /etc/kubernetes/
[root@client ~]# echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
[root@client ~]# source .bash_profile 

```

### 3.4 加载环境变量

```bash
[root@master01 ~]# echo "source <(kubectl completion bash)" >> ~/.bash_profile
[root@master01 ~]# source .bash_profile 

```

## 4. kubectl测试

```bash
[root@client ~]# kubectl get nodes 
[root@client ~]# kubectl get cs
[root@client ~]# kubectl cluster-info 
[root@client ~]# kubectl get po -o wide -n kube-system 

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/o7bgwhnkmk.png)

# 十四、Dashboard搭建

**本节内容都在client节点完成。**

## 1. 下载yaml

```bash
[root@client ~]# wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml

```

如果连接超时，可以多试几次。recommended.yaml已上传，也可以在文末下载。

## 2. 配置yaml

### 2.1 修改镜像地址

```bash
[root@client ~]# sed -i 's/kubernetesui/registry.cn-hangzhou.aliyuncs.com\/loong576/g' recommended.yaml

```

由于默认的镜像仓库网络访问不通，故改成阿里镜像

### 2.2 外网访问

```bash
[root@client ~]# sed -i '/targetPort: 8443/a\ \ \ \ \ \ nodePort: 30001\n\ \ type: NodePort' recommended.yaml

```

配置NodePort，外部通过https://NodeIp:NodePort 访问Dashboard，此时端口为30001

### 2.3 新增管理员帐号

```bash
[root@client ~]# cat >> recommended.yaml << EOF
---
# ------------------- dashboard-admin ------------------- #
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

![图片.png](https://ask.qcloudimg.com/draft/6211241/sivm6c9kma.png)

创建超级管理员的账号用于登录Dashboard

## 3. 部署访问

### 3.1 部署Dashboard

```bash
[root@client ~]# kubectl apply -f recommended.yaml

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/nhi31hx0o5.png)

### 3.2 状态查看

```bash
[root@client ~]# kubectl get all -n kubernetes-dashboard 

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/yyawzblg5i.png)

### 3.3 令牌查看

```bash
[root@client ~]# kubectl describe secrets -n kubernetes-dashboard dashboard-admin

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/nalvqj0k0f.png) 
令牌为：

```bash
eyJhbGciOiJSUzI1NiIsImtpZCI6Ii1SOU1pNGswQnJCVUtCaks2TlBnMGxUdGRSdTlPS0s0MjNjUkdlNzFRVXMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tbXRuZ3giLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNWVjOTdkNzItZTgwZi00MDE2LTk2NTEtZDhkMTYwOGJkODViIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmRhc2hib2FyZC1hZG1pbiJ9.WJPzxkAGYjtq556d3HuXNh6g0sDYm2h6U_FsPDvvfhquYSccPGJ1UzX-lKxhPYyCegc603D7yFCc9zQOzpONttkue3rGdOz8KePOAHCUX7Xp_yTcJg15BPxQDDny6Lebu0fFXh_fpbU2_35nG28lRjiwKG3mV3O5uHdX5nk500RBmLkw3F054ww66hgFBfTH2HVDi1jOlAKWC0xatdxuqp2JkMqiBCZ_8Zwhi66EQYAMT1xu8Sn5-ur_6QsgaNNYhCeNxqHUiEFIZdLNu8QAnsKJJuhxxXd2KhIF6dwMvvOPG1djKCKSyNRn-SGILDucu1_6FoBG1DiNcIr90cPAtA

```

### 3.4 访问

请使用**火狐浏览器**访问：https://control plane ip:30001，即https://172.27.34.35/36/37:30001/
![图片.png](https://ask.qcloudimg.com/draft/6211241/vd8sg1uuz6.png)

![图片.png](https://ask.qcloudimg.com/draft/6211241/m873bh3d6e.png)

接受风险
![图片.png](https://ask.qcloudimg.com/draft/6211241/4irinwr9ys.png)
通过令牌方式登录
![图片.png](https://ask.qcloudimg.com/draft/6211241/6u38nz7x80.png)

登录的首页显示

![图片.png](https://ask.qcloudimg.com/draft/6211241/8fw0sxl6ho.png)

切换到命名空间kubernetes-dashboard，查看资源。

Dashboard提供了可以实现集群管理、工作负载、服务发现和负载均衡、存储、字典配置、日志视图等功能。

为了丰富dashboard的统计数据和图表，可以安装heapster组件。**heapster组件实践详见：**[k8s实践(十一)：heapster+influxdb+grafana实现kubernetes集群监](https://blog.51cto.com/3241766/2448881)

# 十五、k8s集群高可用测试

## 1. 组件所在节点查看

通过ipvsadm查看apiserver所在节点，通过leader-elect查看scheduler和controller-manager所在节点：

### 1.1 apiserver节点查看

在lvs-keepalived01上执行ipvsadm查看apiserver转发到的服务器

```bash
[root@lvs-keepalived01 ~]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.27.34.222:6443 wrr
  -> 172.27.34.35:6443            Route   10     2          0         
  -> 172.27.34.36:6443            Route   10     2          0         
  -> 172.27.34.37:6443            Route   10     2          0  

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/2cc6hpjrh9.png)

### 1.2 controller-manager和scheduler节点查看

在client节点上查看controller-manager和scheduler组件所在节点

```bash
[root@client ~]# kubectl get endpoints kube-controller-manager -n kube-system -o yaml |grep holderIdentity
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"master01_0a2bcea9-d17e-405b-8b28-5059ca434144","leaseDurationSeconds":15,"acquireTime":"2020-01-19T03:07:51Z","renewTime":"2020-01-19T04:40:20Z","leaderTransitions":2}'
[root@client ~]# kubectl get endpoints kube-scheduler -n kube-system -o yaml |grep holderIdentity
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"master01_c284cee8-57cf-46e7-a578-6c0a10aedb37","leaseDurationSeconds":15,"acquireTime":"2020-01-19T03:07:51Z","renewTime":"2020-01-19T04:40:30Z","leaderTransitions":2}'

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/nxoo0lffrh.png)

|       组件名       |           所在节点           |
| :----------------: | :--------------------------: |
|     apiserver      | master01、master02、master03 |
| controller-manager |           master01           |
|     scheduler      |           master01           |

## 2. master01关机

### 2.1 关闭master01

关闭master01，模拟宕机

```bash
[root@master01 ~]# init 0

```

### 2.2 apiserver组件节点查看

lvs-keepalived01上查看apiserver节点链接情况

```bash
[root@lvs-keepalived01 ~]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.27.34.222:6443 wrr
  -> 172.27.34.36:6443            Route   10     4          0         
  -> 172.27.34.37:6443            Route   10     2          0 

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/zezue0uxfy.png)

发现master01的apiserver被移除集群，即访问172.27.34.222:64443时不会被调度到master01

### 2.3 controller-manager和scheduler组件节点查看

client节点上再次运行查看controller-manager和scheduler命令

```bash
[root@client ~]# kubectl get endpoints kube-controller-manager -n kube-system -o yaml |grep holderIdentity
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"master03_9481b109-f236-432a-a2cb-8d0c27417396","leaseDurationSeconds":15,"acquireTime":"2020-01-19T04:42:22Z","renewTime":"2020-01-19T04:45:45Z","leaderTransitions":3}'
[root@client ~]# kubectl get endpoints kube-scheduler -n kube-system -o yaml |grep holderIdentity
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"master03_6d84981b-3ab9-4a00-a86a-47bd2f5c7729","leaseDurationSeconds":15,"acquireTime":"2020-01-19T04:42:23Z","renewTime":"2020-01-19T04:45:48Z","leaderTransitions":3}'
[root@client ~]# 

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/j2t09i53xn.png)

controller-manager和scheduler都被切换到master03节点

|       组件名       |      所在节点      |
| :----------------: | :----------------: |
|     apiserver      | master02、master03 |
| controller-manager |      master03      |
|     scheduler      |      master03      |

### 2.4 集群功能性测试

**所有功能性测试都在client节点完成。**

#### 2.4.1 查询

```bash
[root@client ~]# kubectl get nodes
NAME       STATUS     ROLES    AGE   VERSION
master01   NotReady   master   22h   v1.16.4
master02   Ready      master   22h   v1.16.4
master03   Ready      master   22h   v1.16.4
work01     Ready      <none>   22h   v1.16.4
work02     Ready      <none>   22h   v1.16.4
work03     Ready      <none>   22h   v1.16.4

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/7hif2a5do7.png)

master01状态为NotReady

#### 2.4.2 新建pod

```bash
[root@client ~]# more nginx-master.yaml 
apiVersion: apps/v1             #描述文件遵循extensions/v1beta1版本的Kubernetes API
kind: Deployment                #创建资源类型为Deployment
metadata:                       #该资源元数据
  name: nginx-master            #Deployment名称
spec:                           #Deployment的规格说明
  selector:
    matchLabels:
      app: nginx 
  replicas: 3                   #指定副本数为3
  template:                     #定义Pod的模板
    metadata:                   #定义Pod的元数据
      labels:                   #定义label（标签）
        app: nginx              #label的key和value分别为app和nginx
    spec:                       #Pod的规格说明
      containers:               
      - name: nginx             #容器的名称
        image: nginx:latest     #创建容器所使用的镜像
[root@client ~]# kubectl apply -f nginx-master.yaml 
deployment.apps/nginx-master created
[root@client ~]# kubectl get po -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
nginx-master-75b7bfdb6b-9d66p   1/1     Running   0          20s   10.244.3.6   work01   <none>           <none>
nginx-master-75b7bfdb6b-h4bql   1/1     Running   0          20s   10.244.5.5   work03   <none>           <none>
nginx-master-75b7bfdb6b-zmc68   1/1     Running   0          20s   10.244.4.5   work02   <none>           <none>
```

![图片.png](https://ask.qcloudimg.com/draft/6211241/uujbw83kjb.png)

以新建pod nginx为例测试集群是否能正常对外提供服务。

### 2.5 结论

在3节点的k8s集群中，当有一个control plane节点宕机时，集群各项功能不受影响。

## 3. master02关机

在master01处于关闭状态下，继续关闭master02，测试集群还能否正常对外服务。

### 3.1 关闭master02

```bash
[root@master02 ~]# init 0

```

### 3.2 apiserver组件节点查看

```bash
[root@lvs-keepalived01 ~]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.27.34.222:6443 wrr
  -> 172.27.34.37:6443            Route   10     6          20 

```

![图片.png](https://ask.qcloudimg.com/draft/6211241/dfh4lxns2m.png)

此时对集群的访问都转到master03

### 3.3 集群功能测试

```bash
[root@client ~]# kubectl get nodes
The connection to the server 172.27.34.222:6443 was refused - did you specify the right host or port?

```

### 3.4 结论

在3节点的k8s集群中，当有两个control plane节点同时宕机时，etcd集群崩溃，整个k8s集群也不能正常对外服务。

#  十六、lvs-keepalived集群高可用测试

## 1. 高可用测试前检查

### 1.1 k8s集群检查

```bash
[root@client ~]# kubectl get nodes
NAME       STATUS   ROLES    AGE    VERSION
master01   Ready    master   161m   v1.16.4
master02   Ready    master   144m   v1.16.4
master03   Ready    master   142m   v1.16.4
work01     Ready    <none>   137m   v1.16.4
work02     Ready    <none>   135m   v1.16.4
work03     Ready    <none>   134m   v1.16.4

```

集群内个节点运行正常

### 1.2 vip查看

```bash
[root@lvs-keepalived01 ~]# ip a|grep 222
    inet 172.27.34.222/32 scope global ens160

```

发现vip运行在lvs-keepalived01上

### 1.3 链接情况

**lvs-keepalived01:**

```bash
[root@lvs-keepalived01 ~]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.27.34.222:6443 wrr
  -> 172.27.34.35:6443            Route   10     6          0         
  -> 172.27.34.36:6443            Route   10     0          0         
  -> 172.27.34.37:6443            Route   10     38         0  

```

**lvs-keepalived02:**

```bash
[root@lvs-keepalived02 ~]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.27.34.222:6443 wrr
  -> 172.27.34.35:6443            Route   10     0          0         
  -> 172.27.34.36:6443            Route   10     0          0         
  -> 172.27.34.37:6443            Route   10     0          0  

```

## 2. lvs-keepalived01关机

关闭lvs-keepalived01，模拟宕机

```bash
[root@lvs-keepalived01 ~]# init 0

```

### 2.1 k8s集群检查

```bash
[root@client ~]# kubectl get nodes
NAME       STATUS   ROLES    AGE    VERSION
master01   Ready    master   166m   v1.16.4
master02   Ready    master   148m   v1.16.4
master03   Ready    master   146m   v1.16.4
work01     Ready    <none>   141m   v1.16.4
work02     Ready    <none>   139m   v1.16.4
work03     Ready    <none>   138m   v1.16.4

```

集群内个节点运行正常

### 2.2 vip查看

```bash
[root@lvs-keepalived02 ~]# ip a|grep 222
    inet 172.27.34.222/32 scope global ens160

```

发现vip已漂移至lvs-keepalived02

### 2.3 链接情况

**lvs-keepalived02:**

```bash
[root@lvs-keepalived02 ~]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.27.34.222:6443 wrr
  -> 172.27.34.35:6443            Route   10     1          0         
  -> 172.27.34.36:6443            Route   10     4          0         
  -> 172.27.34.37:6443            Route   10     1          0  

```

### 2.4 集群功能性测试

```bash
[root@client ~]# kubectl delete -f nginx-master.yaml 
deployment.apps "nginx-master" deleted
[root@client ~]# kubectl get po -o wide
NAME                            READY   STATUS        RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
nginx-master-75b7bfdb6b-9d66p   0/1     Terminating   0          20m   10.244.3.6   work01   <none>           <none>
nginx-master-75b7bfdb6b-h4bql   0/1     Terminating   0          20m   10.244.5.5   work03   <none>           <none>
nginx-master-75b7bfdb6b-zmc68   0/1     Terminating   0          20m   10.244.4.5   work02   <none>           <none>
[root@client ~]# kubectl get po -o wide
No resources found in default namespace.
```

![图片.png](https://ask.qcloudimg.com/draft/6211241/ipwk4mtog2.png)

删除之前新建的pod nginx，成功删除。

### 2.5 结论

当lvs-keepalived集群有一台宕机时，对k8s集群无影响，仍能正常对外提供服务。
