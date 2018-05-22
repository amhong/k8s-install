# kube
## 一、安装前的准备
1、安装操作系统时删除swap分区，若已经安装操作系统需禁用swap
2、关闭防火墙
```bash
$ systemctl stop firewalld.service
$ systemctl disable firewalld.service
```
3、关闭SELinux
```bash
$ vi /etc/selinux/config
修改"SELINUX=disabled"
```

## 二、安装 Docker
### 2.1 如果有旧版本先卸载旧版本
```bash
$ sudo apt-get remove docker docker-engine docker.io
