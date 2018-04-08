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

如果kubeadm 初始化失败，为了清除掉其他的数据，通过

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
使用 `kubectl get pod -n kube-system` 来查看系统pod是否初始化成功了. <br>

***


