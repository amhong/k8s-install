# kube
## 一、安装前的准备
1、安装操作系统时删除swap分区，若已经安装操作系统需禁用swap

2、修改每台服务器的主机名和IP地址。

3、如果是虚拟机还要保证每个虚拟机的MAC地址和产品UUID不重复，使用如下两条命令查看
```bash
ifconfig -a
cat /sys/class/dmi/id/product_uuid
```
4、关闭防火墙
```bash
systemctl stop firewalld.service
systemctl disable firewalld.service
```
5、关闭SELinux
```bash
vi /etc/selinux/config
修改"SELINUX=disabled"
```
6、配置host，使每个Node都可以通过名字解析到ip地址
```bash
vi /etc/hosts
加入如下片段(ip地址和servername替换成自己的)
192.168.1.201 kube-1
192.168.1.202 kube-2
192.168.1.203 kube-3
```
7、接受所有ip的数据包转发
```bash
vi /lib/systemd/system/docker.service
找到ExecStart=xxx，在这行上面加入一行，内容如下：(k8s的网络需要)
ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
```
8、设置系统参数 - 允许路由转发，不对bridge的数据进行处理
```bash
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
9、重启
```bash
shutdown -r now
```

## 二、安装 Docker
### 2.1 如果有旧版本先卸载旧版本
```bash
sudo apt-get remove docker docker-engine docker.io
```
### 2.2 安装 Docker-ce
```bash
# 更新软件包
yum update
# 安装必要的一些系统工具
yum install -y yum-utils device-mapper-persistent-data lvm2
# 添加软件源信息
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 更新并安装 Docker-CE
yum makecache fast
sudo yum -y install docker-ce
# 修改 Docker 镜像仓库源
mkdir /etc/docker
vi /etc/docker/daemon.json
{
  "registry-mirrors": ["https://of1ooeiv.mirror.aliyuncs.com"]
}
# 开启Docker服务
systemctl enable docker
systemctl start docker
```
## 三、安装 etcd
### 3.1 创建 etcd CA 证书
1、Install cfssl and cfssljson:
```bash
curl -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
curl -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x /usr/local/bin/cfssl*
```
2、
```bash
mkdir -p /etc/kubernetes/pki/etcd
cd /etc/kubernetes/pki/etcd
cat <<EOF > ca-config.json
   {
       "signing": {
           "default": {
               "expiry": "43800h"
           },
           "profiles": {
               "server": {
                   "expiry": "43800h",
                   "usages": [
                       "signing",
                       "key encipherment",
                       "server auth",
                       "client auth"
                   ]
               },
               "client": {
                   "expiry": "43800h",
                   "usages": [
                       "signing",
                       "key encipherment",
                       "client auth"
                   ]
               },
               "peer": {
                   "expiry": "43800h",
                   "usages": [
                       "signing",
                       "key encipherment",
                       "server auth",
                       "client auth"
                   ]
               }
           }
       }
   }
   EOF

## 四、安装 kubernetes
### 4.1 安装kubelet、kubectl、kubeadm
```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet
```
### 4.2 验证 cgroup 一致性
```bash
使用 docker info | grep -i cgroup 查看Docker 的 cgroup
使用 cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf 查看 kubeadm 使用的 cgroup
如果 KUBELET_CGROUP_ARGS=--cgroup-driver= 的值与 docker info | grep -i cgroup 的值不一致，则使用以下命令修正
sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```
### 4.3 重启 kubelet
```bash
systemctl daemon-reload
systemctl restart kubelet
```
