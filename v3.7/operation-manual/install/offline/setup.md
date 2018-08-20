--- 
title: 离线安装 
summary: 离线安装云帮
toc: true 
---

离线安装对系统环境要求比较严格，准备工作建议参考执行安装前准备和离线准备工作。

## 一、制作离线包

需要在有网的环境下，且已经安装了docker，且有权限读写/opt/rainbond目录。

```bash
# 默认会在/opt/目录下创建rainbond相关目录文件
curl https://raw.githubusercontent.com/goodrain/rainbond-install/v3.7/scripts/offline.sh -o ./offline.sh
chmod +x offline.sh
./offline.sh
```

## 二、准备工作

1. 将离线包同步到离线环境 `tar xf install.offline.v3.7.0.xxxx.tgz -C /`。
2. 请提前安装好如下包,避免安装失败。

```bash
# 通用软件工具包
tar ntpdate wget curl tree lsof htop nload net-tools telnet rsync git dstat iotop lvm2 pwgen
# centos
nfs-utils perl bind-utils iproute bash-completion createrepo
# debian/ubuntu
nfs-kernel-server nfs-common dnsutils python-pip python-apt apt-transport-https uuid-runtime iproute2 systemd
```

## 三、安装Rainbond

```bash
cd /opt/rainbond/install
./grctl init --install-type offline
```

## 四、安装提示报错

如果是安装包失败或者提示什么命令未找到，请停止安装,根据提示自行下载相关安装包

```
mkdir -p /root/pkgs
# centos
yum install <包名> --downloadonly --downloaddir=/root/pkgs
# debian,deb默认放在/var/cache/apt/archives/目录下
apt install <包名> -d -y
```

手动安装缺失的包或工具,然后重新执行安装。

## 四、安装问题建议

当离线安装Rainbond 遇到问题时，请参考本篇相关文档。如果问题未解决，请按文档要求收集相关信息通过 Github [提供给 Rainbond开发者](https://github.com/goodrain/rainbond/issues/new)。

1. 系统版本
2. 机器配置
3. 云帮版本
4. 具体什么报错(最好有必要的图文说明)
5. 是否重新执行安装
6. 是否采用了上述的解决方案