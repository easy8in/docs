# Docker 文档

## 简介

Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中,然后发布到任何流行的 [Linux](https://baike.baidu.com/item/Linux) 机器或 Windows 机器上,也可以实现虚拟化,容器是完全使用沙箱机制,相互之间不会有任何接口。Docker 有两个版本社区本版与企业版，企业版本是收费产品。我们一般选择社区版就够用了。

### 完整的 Docker 由以下四部分组成：

1. Docker Client 客户端
2. Docker Daemon 守护进程
3. Docker Image 镜像
4. Docker Container 容器

## Docker 架构

Docker 使用客户端 - 服务器 (C/S) 架构模式，使用远程 API 来管理和创建 Docker 容器。Docker 容器通过 Docker 镜像来创建。容器与镜像的关系类似于面向对象编程中的对象与类。

| Docker          | 面向对象              |
| --------------- | ---------------------|
| 镜像 (images)    | 类 (class)             |
| 容器 (container) | 对象实例 (new Class()) |

![](docker/docker.png.jpg)

Docker 采用 C/S 架构 Docker daemon 作为服务端接受来自客户的请求，并处理这些请求（创建、运行、分发容器）。 客户端和服务端既可以运行在一个机器上，也可通过 socket 或者 container API 来进行通信。

![](docker/image-20200331084339909.png)

Docker daemon 一般在宿主主机后台运行，等待接收来自客户端的消息。 Docker 客户端则为用户提供一系列可执行命令，用户用这些命令实现跟 Docker daemon 交互。

### 特性

Docker 主要特性以及目标，是 Docker 简化工作中常见问题环境

- Automating the packaging and deployment of applications
	使应用的打包与部署自动化
- Creation of lightweight, private PAAS environments
	创建轻量、私密的 PAAS 环境
- Automated testing and continuous integration/deployment
	实现自动化测试和持续的集成/部署
- Deploying and scaling web apps, databases and backend services 部署与扩展 webapp、数据库和后台服务

### 局限

Docker 并不是全能的，设计之初也不是 KVM 之类虚拟化手段的替代品，简单总结几点：

1. Docker 是基于 Linux 64bit 的，无法在 32bit 的 linux/Windows/unix 环境下使用
2. LXC 是基于 cgroup 等 linux kernel 功能的，因此 container 的 guest 系统只能是 linux base 的
3. 隔离性相比 KVM 之类的虚拟化方案还是有些欠缺，所有 container 公用一部分的运行库
4. 网络管理相对简单，主要是基于 namespace 隔离
5. cgroup 的 cpu 和 cpuset 提供的 cpu 功能相比 KVM 的等虚拟化方案相比难以度量
6. Docker 对 disk 的管理比较有限
7. container 随着用户进程的停止而销毁，container 中的 log 等用户数据不便收集

### App to Docker

在企业用户准备把应用程序迁往容器之前，理解应用程序的迁移过程是非常重要的。这里将介绍把用户应用程序迁往 Docker 容器的五个基本步骤。

- 增加代码。为了创建镜像，企业用户需要使用一个 Dockerfile 来定义映像开发的必要步骤。一旦创建了映像，企业用户就应将其添加至 Docker Hub
- 选择基础映像。当执行应用程序迁移时，应尽量避免推倒重来的做法。搜索 Docker 注册库找到一个基本的 Docker 映像并将其作为应用程序的基础来使用。例如编写 Dockerfile 首页行加入 FROM tomcat:latest。
- 配置测试部署。应对在容器中运行的应用程序进行配置，以便于让应用程序知道可以在哪里连接外部资源或者应用程序集群中的其他容器。企业用户可以把这些配置部署在容器中或使用环境变量。

## 安装 Docker

### 官方快速安装脚本

```bash
curl -o- https://get.docker.com | bash --mirror Aliyun
```

### CentOS

#### 如果系统中已经安装过，可以使用如下命令卸载掉老版本

```bash
sudo yum remove docker \
			  docker-client \
			  docker-client-latest \
			  docker-common \
			  docker-latest \
			  docker-latest-logrotate \
			  docker-logrotate \
			  docker-engine
```

> [!tip]
> 1. Docker 的默认安装位置 /var/lib/docker/，包含（images, containers, volumes, networks）
> 2. 社区版最新版本，已经将 docker-engine 改名为 docker-ce

#### repository 配置

1. 安装所需的 package，yum-utils 提供 yum-config-manager 和 device-mapper-persistent-data 工具，devicemapper 使用需要 lvm2 的支持。

```bash
$ sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```

1. 使用 yum-config-manager 配置 repository

```bash
$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

#### 安装社区版 Docker Engine

1. 安装 latest 版本 docker-engine

```bash
$ sudo yum install docker-ce docker-ce-cli containerd.io
```

1. 安装指定版本的 docker

```bash
# 查看docker版本
yum list docker-ce --showduplicates | sort -r

#安装指定版本
sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
```

#### 启动 Docker Engine

```bash
$ sudo systemctl start docker
```

#### 验证 Docker Engine 是否安装成功

```bash
$ sudo docker run hello-world
```

### Ubuntu

#### 系统要求（64-bit version）

  Disco 19.04

  Cosmic 18.10

  Bionic 18.04 (LTS)

  Xenial 16.04 (LTS)

#### 卸载老版本 Docker

  ```bash
  sudo apt-get remove docker docker-engine docker.io containerd runc
  ```

> [!tip]
> 老版本 Docker engine 叫做 docker-engine，最新版本已经改名为 docker-ce

#### 配置 repository

```bash
sudo apt-get update

sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
    
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

#### 安装

```bash
$ sudo apt-get update

$ sudo apt-get install docker-ce docker-ce-cli containerd.io

$ sudo docker run hello-world
```

#### 参考地址

[docker 官方网站](https://docs.docker.com/)

[docker centos 安装文档](https://docs.docker.com/install/linux/docker-ce/centos/)

[docker 菜鸟教程](https://www.runoob.com/docker/docker-tutorial.html)

### docker 镜像加速​

在 aliyun 中搜索镜像,在搜索结果中找到 <容器镜像服务> ,在操作面板中找到<镜像加速器> 按文档操作

## Docker 命令

### 生命周期管理

| 命令 | 格式 |

| ------------ | ----------------------- | --- |

| 启动 Docker 服务 | systemctl start docker |

| 停止 Docker 服务 | systemctl stop docker |

| 重启 Docker 服务 | systemctl restart docker |

| 查看 Docker 服务状态 | systemctl status docker |

| 加入开机启动 | systemctl enable docker |

### Docker 镜像相关命令

| 命令 | 格式          |
| ---- | ------------- |
| 查看 | docker images |
| 删除 | docker rmi {}|
