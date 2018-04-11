# Kubernetes墙内安装

## 前期准备
环境：阿里云CentOS7.4 <br>
关闭防火墙 <br>
```shell
systemctl disable firewalld
systemctl stop firewalld
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
```

## 下载指定版本Docker
因为打算装K8s 1.7.5, 需要Docker的版本是1.12.6, 所以需要安装指定版本的Docker. <br>

添加软件源 <br>
```
cat >/etc/yum.repos.d/docker.repo <<EOF
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
```
更新软件源

`yum makecache`

显示软件源中所有Docker软件包安装信息

`yum provides docker-engine`

之后在里面挑选你想要装的docker版本 

`yum install docker-engine-1.12.6-1.el7.centos.x86_64`

之后 `systemctl start docker` 来开启Docker服务 <br>
 
## Docker加速
之后配置Docker加速器，使Docker拉镜像更快. <br>
此处选择阿里云的镜像加速器，在阿里云的容器镜像服务中可以找到. 此处不再赘述.<br>

## 安装Kubernetes套件

kubernetes yum源
```shell
cat >> /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
EOF
```
安装
```shell
yum install -y kubernetes-cni-0.5.1-0.x86_64 
yum install -y kubelet-1.7.5-0.x86_64 
yum install -y kubectl-1.7.5-0.x86_64 
yum install -y kubeadm-1.7.5-0.x86_64
```

## 下载K8s运行所需镜像
在docker官方镜像仓库中发现了一个维护的很好的[k8s镜像仓库](https://hub.docker.com/u/mirrorgooglecontainers/),写了一段代码来拉镜像
```shell
images=(etcd-amd64:3.0.17 pause-amd64:3.0 kube-proxy-amd64:v1.7.5 kube-scheduler-amd64:v1.7.5 kube-controller-manager-amd64:v1.7.5 kube-apiserver-amd64:v1.7.5 kubernetes-dashboard-amd64:v1.6.1 k8s-dns-sidecar-amd64:1.14.4 k8s-dns-kube-dns-amd64:1.14.4 k8s-dns-dnsmasq-nanny-amd64:1.14.4)
for imageName in ${images[@]} ; do
  docker pull mirrorgooglecontainers/$imageName
  docker tag mirrorgooglecontainers/$imageName gcr.io/google_containers/$imageName
  docker rmi mirrorgooglecontainers/$imageName
done
```

## 配置kubelet

配置pod基础镜像
```shell
cat > /etc/systemd/system/kubelet.service.d/20-pod-infra-image.conf <<EOF
[Service]
Environment="KUBELET_EXTRA_ARGS=--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/szss_k8s/pause-amd64:3.0"
EOF
```

配置kubelet cgroups 为 cgroupfs(`安装docker 1.12.6及版本需要设置kubelet的cgroup-driver=cgroupfs`)
```shell
sed -i 's/cgroup-driver=systemd/cgroup-driver=cgroupfs/g' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```
之后重启docker 与 kubelet.
```shell
systemctl enable docker
systemctl enable kubelet
systemctl start docker
systemctl start kubelet
```

## kubeadm创建集群
此处使用了别人上传好的etcd镜像, --pod-network-cidr不需要修改.<br>
```
export KUBE_REPO_PREFIX="registry.cn-hangzhou.aliyuncs.com/szss_k8s"
export KUBE_ETCD_IMAGE="registry.cn-hangzhou.aliyuncs.com/szss_k8s/etcd-amd64:3.0.17"
kubeadm init  --kubernetes-version=v1.7.2 --pod-network-cidr=10.244.0.0/12
```

若过程中卡在了
`[apiclient] Created API client, waiting for the control plane to become ready`<br>
则先耐心等一会儿，若超过5min还不可以应该是因为kubelet与docker的cgroups不统一导致的. <br>
此处使用 systemctl status kubelet 与 journalctl 来查看报错.<br>
在[之前写的一个博客](https://humbertzhang.github.io/2018/02/21/%E5%88%A9%E7%94%A8Kubeadm%E5%90%91Kubernetes%E9%9B%86%E7%BE%A4%E5%8A%A0%E5%85%A5%E8%8A%82%E7%82%B9/)的Step3中简述了如何解决这个问题。

如果kubeadm 初始化失败，为了清除掉其他的数据，通过 `kubeadm reset` 和 重启kubelet来重置。

## 配置kubectl的kubeconfig
```shell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

## 在Master节点配置网络 flannel
```shell
kubectl --namespace kube-system apply -f https://raw.githubusercontent.com/coreos/flannel/v0.8.0/Documentation/kube-flannel-rbac.yml
rm -rf kube-flannel.yml 
wget https://raw.githubusercontent.com/coreos/flannel/v0.8.0/Documentation/kube-flannel.yml
sed -i 's/quay.io\/coreos\/flannel:v0.8.0-amd64/registry.cn-hangzhou.aliyuncs.com\/szss_k8s\/flannel:v0.8.0-amd64/g' ./kube-flannel.yml
kubectl --namespace kube-system apply -f ./kube-flannel.yml
```
之后可以通过`kubectl get nodes`来查看集群初始化是否成功.<br>
使用 `kubectl get pod -n kube-system` 来查看各个系统pod是否初始化成功. <br>

***
这里说一下其他的一些经验。

## 干掉阿里云监控
因为阿里云的服务器上都是有监控进程的，如果你做科学上网之类的事情很可能就会被打电话警告(亲身经历 = =),所以如果有翻墙的需求的话，一定要提前把阿里云的监控干掉。

#### 卸载阿里云盾监控
```
wget http://update.aegis.aliyun.com/download/uninstall.sh
chmod +x uninstall.sh
./uninstall.sh

wget http://update.aegis.aliyun.com/download/quartz_uninstall.sh
chmod +x quartz_uninstall.sh
./quartz_uninstall.sh
```

#### 删除残留
```
pkill aliyun-service
rm -fr /etc/init.d/agentwatch /usr/sbin/aliyun-service
rm -rf /usr/local/aegis*
```

#### 屏蔽云盾 IP
```
iptables -I INPUT -s 140.205.201.0/28 -j DROP
iptables -I INPUT -s 140.205.201.16/29 -j DROP
iptables -I INPUT -s 140.205.201.32/28 -j DROP
iptables -I INPUT -s 140.205.225.192/29 -j DROP
iptables -I INPUT -s 140.205.225.200/30 -j DROP
iptables -I INPUT -s 140.205.225.184/29 -j DROP
iptables -I INPUT -s 140.205.225.183/32 -j DROP
iptables -I INPUT -s 140.205.225.206/32 -j DROP
iptables -I INPUT -s 140.205.225.205/32 -j DROP
iptables -I INPUT -s 140.205.225.195/32 -j DROP
iptables -I INPUT -s 140.205.225.204/32 -j DROP
```

## 将socks5代理转换为http代理，为go get提供支持
因为想要用Golang对k8s做一下二次而发，所以就需要go get 来下载一些包，而go get走的是http代理，并不是socks5代理，因此需要将socks5代理转换为http代理来为其提供支持。此处采用privoxy来简单地将socks5的代理转换为http代理。
安装privoxy <br>
`yum install privoxy`  <br>
配置privoxy <br> 
`vim /etc/privoxy/config` <br>
添加下面这一行 <br>
`forward-socks5 / 127.0.0.1:1080 ` <br>
forward-socks5代表转发到socks5代理，/代表所有的URL都转发(也可以在这里写url patten)，127.0.0.1:1080是socks代理地址.

启动privoxy <br>

`/etc/init.d/privoxy restart`

服务默认监听在本地127.0.0.1:8118上 <br>

***

Reference: <br>
+ http://www.yunweipai.com/archives/20884.html 
+ http://www.ttlsa.com/linux/privoxy-convert-socks-proxy-to-http/
+ https://github.com/ssrpanel/SSRPanel/wiki/%E5%8D%B8%E8%BD%BD%E9%98%BF%E9%87%8C%E4%BA%91%E7%9B%BE%E7%9B%91%E6%8E%A7&%E5%B1%8F%E8%94%BD%E4%BA%91%E7%9B%BEIP#%E5%8D%B8%E8%BD%BD%E9%98%BF%E9%87%8C%E4%BA%91%E7%9B%BE%E7%9B%91%E6%8E%A7
+ http://zxc0328.github.io/2017/10/26/k8s-setup-1-7/#more
+ https://blog.jsjs.org/?p=414
