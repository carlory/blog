---
title: Setting up a local mirror for Docker Hub images
date: 2019-07-15 00:21:07
tags:
- registry
---

## 环境准备

### 实施环境

| 角色         | 主机         | 配置 | 备注 |      |
| :----------- | :----------- | :--- | :--- | ---- |
| 镜像加速节点 | 192.168.2.79 | 1. /data/registry目录300G的存储</br>2. 机器配置4核8G</br>3. 开放端口为443 | 此节点用来部署docker-registry容器，提供镜像加速的功能。对外访问的域名为mirror.regsitry.pelican.local |      |
| 代理 |  | http://192.168.2.30:8080 | 企业内网环境下，出于安全方面的考虑，数据中心只能通过代理才可以访问到外部网络。通过此代理可以访问到以下域名：</br>1. registry-1.docker.io:443</br>2. auth.docker.io:443</br>3. production.cloudflare.docker.com:443 | |

### 环境检查

#### 检查节点的系统配置

##### 检查存储镜像的目录容量

执行下面命令，检查/data/registry目录的容量大小，期望容量为300G。

```shell
df -h
```

##### 检查CPU和内存的配置

执行下面命令，检查CPU和内存的配置，期望配置为4核8G。

```shell
lscpu
free -mh
```

##### 检查443端口是否被占用

执行下面命令，查看端口占用情况，期望结果为443端口未被占用。

```shell
ss -lnp | grep 443
```

##### 检查yum源

执行下面命令，查看yum源是否可以安装docker。

```shell
yum list | grep docker 
```

##### 检查代理

执行下面命令，检查通过提供的代理，是否可以访问下面域名：

```shell
# 设置代理（当前shell生效）
export http_proxy=http://192.168.2.30:8080
export https_proxy=http://192.168.2.30:8080

# 检查在内网环境下，外部域名和端口是否可以访问
curl -I https://registry-1.docker.io
curl -I https://auth.docker.io
curl -I https://production.cloudflare.docker.com

# 清除代理（当前shell）
unset http_proxy
unset https_proxy
```

## 实施步骤

由于实施环境为网内，无法访问外网，需事先下载好安装包，并将安装包导入到镜像加速节点上。下面列举一下，离线安装包的文件组织结构。

```shell
.
├── bin
│   └── docker-compose
├── config
│   └── mirror
│       ├── cert
│       │   ├── mirror.registry.pelican.local.key
│       │   ├── mirror.registry.pelican.local.crt
│       └── mirror_env
├── docker-compose.yaml
├── images
│   └── docker-registry-v2.tar
```

### 安装软件

```shell
# 安装docker
yum install docker

# 设置开机自启并启动docker
systemctl enable docker
systemctl start docker

# 安装docker-compose（拷贝提前下好的docker-compose文件至$PATH目录下）
cp bin/docker-compose /usr/bin/docker-compose
chmod a+x /usr/bin/docker-compose
```

### 创建数据目录

```shell
# 存放镜像
mkdir -p /data/registry

# 存放证书
mkdir -p /data/cert
```

### 拷贝证书

```shell
cp config/mirror/cert/mirror.registry.pelican.local.key /data/cert/mirror.registry.pelican.local.key
cp config/mirror/cert/mirror.registry.pelican.local.crt /data/cert/mirror.registry.pelican.local.crt
```

### 配置环境变量

```shell
REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io
REGISTRY_HTTP_ADDR=0.0.0.0:443
REGISTRY_HTTP_TLS_CERTIFICATE=/etc/cert/server.crt
REGISTRY_HTTP_TLS_KEY=/etc/cert/server.key
http_proxy=http://192.168.2.30:8080
https_proxy=http://192.168.2.30:8080
no_proxy=localhost,127.0.0.1
```

### 配置docker-compose.yaml文件

```yaml
version: '2.3'
services:
  mirror:
    image: registry:2
    container_name: mirror-registry
    restart: always
    volumes:
    - /data/registry:/var/lib/registry:z
    - type: bind
      source: /data/cert/mirror.registry.pelican.local.key
      target: /etc/cert/server.key
    - type: bind
      source: /data/cert/mirror.registry.pelican.local.crt
      target: /etc/cert/server.crt
    ports:
    - 443:443
    env_file:
      ./config/mirror/mirror_env
```

### 部署容器

```shell
# 导入镜像
docker load -i docker-registry-v2.tar

# 启动容器
docker-compose up -d

# 查看状态
watch docker-compose ps
```

## 部署验证

### 更新dockerd配置，并使配置生效

在需要使用镜像加速的节点上,执行下面步骤

```shell
# 添加下面配置
vi /etc/docker/daemon.json

{
  "registry-mirrors": ["https://mirror.registry.pelican.local"],
  "insecure-registries": ["mirror.registry.pelican.local"]
}

# 发送信号，使配置生效
kill -SIGHUP `pidof dockerd-current`

# 查看dockerd日志，当发现下面信息表示配置生效
journalctl -u docker -ef

level=info msg="Got signal to reload configuration, reloading from: /etc/docker/daemon.json"
```

### 拉取busybox镜像，进行验证

```shell
docker pull busybox
```