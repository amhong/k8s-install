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
