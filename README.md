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
### 4.1 安装kube-apiserver
1、生成 kube-apiserver 配置文件（各Master节点）
```bash
cat >/etc/systemd/system/kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
[Service]
ExecStart=/usr/bin/kube-apiserver \
--admission-control=NamespaceLifecycle,LimitRanger,DefaultStorageClass,ResourceQuota,NodeRestriction \
--insecure-bind-address=0.0.0.0 \
--kubelet-https=false \
--service-cluster-ip-range=10.68.0.0/16 \
--service-node-port-range=20000-40000 \
--etcd-servers=http://kube-1:2379,http://kube-2:2379,http://kube-3:2379 \
--enable-swagger-ui=true \
--allow-privileged=true \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/var/lib/audit.log \
--event-ttl=1h \
--v=2
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```
2、启用 kube-apiserver 服务（各Master节点）
```bash
systemctl enable kube-apiserver.service
systemctl start kube-apiserver
```
3、查看 kube-apiserver 日志确认启动状态（各Master节点）
```bash
journalctl -f -u kube-apiserver
```
### 4.2 各Master节点配置 kube-apiserver 高可用
1、安装 keepalived（各Master节点）
```bash
yum install keepalived
```
2、生成 keepalived 配置文件（各Master节点， 和interface）

注意修改 state，Master节点1为 MASTER，其余节点为 BACKUP

注意修改 priority，Master节点1为 100，Master节点2为 99，Master节点2为 98

注意修改 interface 为各节点网卡名称
```bash
cat >/etc/keepalived/keepalived.conf <<EOF
global_defs {
  router_id kubernetes
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    authentication {
        auth_type PASS
        auth_pass 4be37dc3b4c90194d1600c483e10ad1d
    }
    virtual_ipaddress {
        192.168.1.200
    }
}

virtual_server 192.168.1.200 8080 {
    delay_loop 6
    lb_algo wrr
    lb_kind DR
    protocol TCP
    real_server 192.168.1.201 8080 {
        weight 10
        HTTP_GET {
            url {
                path /healthz
                digest 444bcb3a3fcf8389296c49467f27e1d6
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 1
        }
    }
    real_server 192.168.1.202 8080 {
        weight 10
        HTTP_GET {
            url {
                path /healthz
                digest 444bcb3a3fcf8389296c49467f27e1d6
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 1
        }
    }
    real_server 192.168.1.203 8080 {
        weight 10
        HTTP_GET {
            url {
                path /healthz
                digest 444bcb3a3fcf8389296c49467f27e1d6
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 1
        }
    }
}
EOF
```
3、启用 keepalived 服务（各Master节点）
```bash
systemctl enable keepalived
systemctl start keepalived
```
3、查看 keepalived 日志确认启动状态（各Master节点）
```bash
journalctl -f -u keepalived
```

cat >/etc/systemd/system/kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
[Service]
ExecStart=/usr/bin/kube-controller-manager \
--address=127.0.0.1 \
--master=http://127.0.0.1:8080 \
--allocate-node-cidrs=true \
--service-cluster-ip-range=10.68.0.0/16 \
--cluster-cidr=172.20.0.0/16 \
--cluster-name=kubernetes \
--leader-elect=true \
--cluster-signing-cert-file= \
--cluster-signing-key-file= \
--v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF

cat >/etc/systemd/system/kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-scheduler \
--address=127.0.0.1 \
--master=http://127.0.0.1:8080 \
--leader-elect=true \
--v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF

cat >/etc/systemd/system/kube-calico.service <<EOF
[Unit]
Description=calico node
After=docker.service
Requires=docker.service

[Service]
User=root
PermissionsStartOnly=true
ExecStart=/usr/bin/docker run --net=host --privileged --name=calico-node \
-e ETCD_ENDPOINTS=http://kube-1:2379,http://kube-2:2379,http://kube-3:2379 \
-e CALICO_LIBNETWORK_ENABLED=true \
-e CALICO_NETWORKING_BACKEND=bird \
-e CALICO_DISABLE_FILE_LOGGING=true \
-e CALICO_IPV4POOL_CIDR=172.20.0.0/16 \
-e CALICO_IPV4POOL_IPIP=off \
-e FELIX_DEFAULTENDPOINTTOHOSTACTION=ACCEPT \
-e FELIX_IPV6SUPPORT=false \
-e FELIX_LOGSEVERITYSCREEN=info \
-e FELIX_IPINIPMTU=1440 \
-e FELIX_HEALTHENABLED=true \
-e IP= \
-v /var/run/calico:/var/run/calico \
-v /lib/modules:/lib/modules \
-v /run/docker/plugins:/run/docker/plugins \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /var/log/calico:/var/log/calico \
registry.cn-hangzhou.aliyuncs.com/imooc/calico-node:v2.6.2
ExecStop=/usr/bin/docker rm -f calico-node
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

主节点+工作节点：
```bash
mkdir -p /var/lib/kubelet
mkdir -p /etc/kubernetes
mkdir -p /etc/cni/net.d

cat >/etc/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/bin/kubelet \
--address=$PRIVATE_IP \
--hostname-override=$PRIVATE_IP \
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/imooc/pause-amd64:3.0 \
--kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
--network-plugin=cni \
--cni-conf-dir=/etc/cni/net.d \
--cni-bin-dir=/usr/bin \
--cluster-dns=10.68.0.2 \
--cluster-domain=cluster.local. \
--allow-privileged=true \
--fail-swap-on=false \
--logtostderr=true \
--v=2
#kubelet cAdvisor 默认在所有接口监听 4194 端口的请求, 以下iptables限制内网访问
ExecStartPost=/sbin/iptables -A INPUT -s 10.0.0.0/8 -p tcp --dport 4194 -j ACCEPT
ExecStartPost=/sbin/iptables -A INPUT -s 172.16.0.0/12 -p tcp --dport 4194 -j ACCEPT
ExecStartPost=/sbin/iptables -A INPUT -s 192.168.0.0/16 -p tcp --dport 4194 -j ACCEPT
ExecStartPost=/sbin/iptables -A INPUT -p tcp --dport 4194 -j DROP
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

cat >/etc/kubernetes/kubelet.kubeconfig <<EOF
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: http://192.168.1.201:8080
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: ""
  name: system:node:kube-master
current-context: system:node:kube-master
kind: Config
preferences: {}
users: []
EOF

cat >/etc/cni/net.d/10-calico.conf <<EOF
{
    "name": "calico-k8s-network",
    "cniVersion": "0.1.0",
    "type": "calico",
    "etcd_endpoints": "http://kube-1:2379,http://kube-2:2379,http://kube-3:2379",
    "log_level": "info",
    "ipam": {
        "type": "calico-ipam"
    },
    "kubernetes": {
        "k8s_api_root": "http://192.168.1.201:8080"
    }
}
EOF

```
