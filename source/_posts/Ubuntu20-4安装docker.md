---
title: Ubuntu20.4安装docker
categories:
  - - linux
  - - Ubuntu
tags:
  - - linux
date: 2023-02-12 15:05:35
---

在 Ubuntu 上安装 Docker 非常直接。启用 Docker 软件源，导入 GPG key，并且安装软件包。

1. 使用下面的 `curl`  导入源仓库的 GPG key：

```bash
curl -fsSL <https://download.docker.com/linux/ubuntu/gpg> | sudo apt-key add -
```

2. 将 Docker APT 软件源添加到你的系统：

```bash
sudo add-apt-repository "deb [arch=amd64] <https://download.docker.com/linux/ubuntu> $(lsb_release -cs) stable"
```

3. 安装 Docker docker-compose最新版本:

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

一旦安装完成，Docker 服务将会自动启动。你可以输入下面的命令，验证它：

```
sudo systemctl status docker
```

**以非 Root 用户身份执行 Docker**

```
sudo usermod -aG docker $USER
```

然后退出当前用户比如切换为root，再次切换为user。

```sh
sudo su
su dong
```