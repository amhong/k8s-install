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
vi /etc/hosts
加入如下片段(ip地址和servername替换成自己的)
192.168.1.201 kube-1
192.168.1.202 kube-2
192.168.1.203 kube-3

7、重启
```bash
shutdown -r now
```

## 二、安装 Docker
### 2.1 如果有旧版本先卸载旧版本
```bash
sudo apt-get remove docker docker-engine docker.io
```
