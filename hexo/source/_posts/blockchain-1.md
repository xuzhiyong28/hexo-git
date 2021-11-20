---
title: 创建你的第一条substrate链
tags:
  - 区块链
categories:  区块链
description : 创建你的第一条substrate链
date: 2020-10-30 13:53:30
---
### 教程
wiki : https://substrate.dev/docs/zh-CN/tutorials/create-your-first-substrate-chain/interact

教程 : https://zhuanlan.zhihu.com/p/342576492

ubantu系统版本 : ubuntu-20.04.2.0-desktop-amd64.iso

### vbox安装ubantu问题

**ifconfig命令不可用**

```
sudo apt install net-tools
```

**允许root用户登录**

http://www.5sharing.com/m/view.php?aid=1541

**ubantu使用root登录**

ubantu默认不能使用root登录，所以先要设置下root用户

```
sudo passwd root #为root修改密码，第一次安装ubanto时候做
su root #切换到root用户
```

**ubantu使用xshell登录不上**

原因是由于xshell远程连接ubuntu是通过ssh协议的，所以，需要给ubuntu安装ssh服务器。

```
sudo apt-get install openssh-server
sudo service ssh restart #重启服务
```

**Ubantu下root账户无法使用xshell**

```
修改 /etc/ssh/sshd_config 文件把PermitRootLogin Prohibit-password 添加#注释掉
新添加：PermitRootLogin yes
重启ssh服务 /etc/init.d/ssh restart
重新使用root连接
```

**Ubantu开发所有端口**

```
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
iptables -F
iptables-save
apt-get install iptables-persistent
netfilter-persistent save
netfilter-persistent reload
```

**Ubantu在vbox上调整分辨率**

1. 设备 --> 安装增强功能
2. cd /media 目录下执行 sudo sh VBoxLinuxAdditions.run
3. 重启sudo reboot 就可以设置分辨率了

### Substrate安装

**安装最新版NodeJs**

可以参考 : https://developer.aliyun.com/article/760687

```
curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -
sudo apt install nodejs
#验证版本
node --version
npm --version
npm config set registry http://registry.npm.taobao.org/ #设置淘宝镜像
```

**安装yarn**

```
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt update
sudo apt install yarn
```

**安装substrate依赖**

```
curl https://getsubstrate.io -sSf | bash -s -- --fast  #设置--fast可以不设置
```

