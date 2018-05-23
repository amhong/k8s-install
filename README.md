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
### 2.2 安装 Docker-ce（使用17.03版）
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
2、创建 etcd CA 证书
```bash
在 Master1 节点上运行以下命令
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
cat >ca-csr.json <<EOF
{
   "CN": "etcd",
   "key": {
       "algo": "rsa",
       "size": 2048
   }
}
EOF
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
cat >client.json <<EOF
{
    "CN": "client",
    "key": {
        "algo": "ecdsa",
        "size": 256
    }
}
EOF
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client.json | cfssljson -bare client
```
3、在 Master2 和 Master3 节点上运行以下命令将上一步生成的证书拷贝过来
```bash
mkdir -p /etc/kubernetes/pki/etcd
cd /etc/kubernetes/pki/etcd
scp root@<Master1-ip-address>:/etc/kubernetes/pki/etcd/ca.pem .
scp root@<Master1-ip-address>:/etc/kubernetes/pki/etcd/ca-key.pem .
scp root@<Master1-ip-address>:/etc/kubernetes/pki/etcd/client.pem .
scp root@<Master1-ip-address>:/etc/kubernetes/pki/etcd/client-key.pem .
scp root@<Master1-ip-address>:/etc/kubernetes/pki/etcd/ca-config.json .
```
### 3.2 安装 etcd 集群
```bash
curl -sSL https://github.com/coreos/etcd/releases/download/v3.1.15/etcd-v3.1.15-linux-amd64.tar.gz | tar -xzv --strip-components=1 -C /usr/local/bin/
rm -rf etcd--linux-amd64*
```
在每个节点上分别设置etcd节点名称和IP地址变量
```bash
PEER_NAME=etcd节点名称
PRIVATE_IP=IP地址
```
在每个节点上创建 etcd 配置文件并启动 etcd 服务
```bash
cat >/etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos/etcd
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
Restart=always
RestartSec=5s
LimitNOFILE=40000
TimeoutStartSec=0

ExecStart=/usr/local/bin/etcd \
--name $PEER_NAME \
--data-dir /var/lib/etcd \
--initial-advertise-peer-urls http://$PRIVATE_IP:2380 \
--listen-peer-urls http://$PRIVATE_IP:2380 \
--listen-client-urls http://$PRIVATE_IP:2379,http://127.0.0.1:2379 \
--advertise-client-urls http://$PRIVATE_IP:2379 \
--initial-cluster-token etcd-cluster-1 \
--initial-cluster kube-1=http://kube-1:2380,kube-2=http://kube-2:2380,kube-3=http://kube-3:2380 \

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start etcd
systemctl enable etcd

```
## 四、安装 kubernetes
### 4.1 安装kubelet、kubectl、kubeadm
```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
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
### 4.4
```bash
cat >config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
  advertiseAddress: $PRIVATE_IP
etcd:
  endpoints:
  - http://kube-1:2379
  - http://kube-2:2379
  - http://kube-3:2379
networking:
  podSubnet: 10.0.0.0/8
apiServerCertSANs:
- 192.168.1.200
apiServerExtraArgs:
  apiserver-count: "3"
EOF

kubeadm init --config=config.yaml
```
