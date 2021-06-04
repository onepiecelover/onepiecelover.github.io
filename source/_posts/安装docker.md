---
title: 安装docker
date: 2021-06-04 18:51:25
tags: docker,容器
---

### ubuntu环境
#### 最新版本
```bash
sudo apt-get remove docker docker-engine docker.io containerd runc

sudo apt-get update

//
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
    
    
//Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

//Verify that you now have the key with the fingerprint
apt-key fingerprint 0EBFCD88

//amd64
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
   
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io
```

#### 指定版本
```bash
//List the versions available in your repo
apt-cache madison docker-ce

//Install a specific version
sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
```

### centos环境
#### rpm包安装——方式一
```bash
# 下载docker-ce包
wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-18.09.2-3.el7.x86_64.rpm

# docker-ce-cli包
wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-cli-18.09.2-3.el7.x86_64.rpm

# containerd.io包
wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.2-3.3.el7.x86_64.rpm

### 安装rpm包（注意顺序）
yum install docker-ce-cli-18.09.2-3.el7.x86_64.rpm

yum install containerd.io-1.2.2-3.3.el7.x86_64.rpm
yum install docker-ce-18.09.2-3.el7.x86_64.rpm
```

[【参考】https://docs.docker.com/install/linux/docker-ce/centos/](https://docs.docker.com/install/linux/docker-ce/centos/)
[【rpm包地址】https://download.docker.com/linux/centos/7/x86_64/stable/Packages/](https://download.docker.com/linux/centos/7/x86_64/stable/Packages/)

#### rpm包安装——方式二
```bash
wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/xxx.rpm

sudo yum install /path/to/package.rpm
sudo systemctl start docker
```

##### 卸载docker
```bash
sudo yum remove docker-ce
sudo rm -rf /var/lib/docker
```

#### yum安装
```bash
sudo yum install -y yum-utils
sudo yum-config-manager   --add-repo https://download.docker.com/linux/centos/docker-ce.repo

sudo yum makecache fast
sudo yum install docker-ce
sudo systemctl start docker
```
[【参考】https://docs.docker.com/engine/installation/linux/centos/#install-using-the-repository](https://docs.docker.com/engine/installation/linux/centos/#install-using-the-repository)
