# Dockerfile 的最佳实践 ｜ Dockerfile 你写的都对么？

随着应用的容器化、上云后，将伴随着 Docker 镜像的构建，构建 Docker 镜像成为了最基本的一步，其中 Dockerfile 便是用来构建镜像的一种文本文件，镜像的优劣全靠 Dockerfile 编写的是否合理、合规。本文将讲述编写 Dockerfile 的一些最佳实践和技巧，让我们的镜像更小、更优。

## 1、Docker 镜像是如何工作的

首先，我们一起回顾下 Docker 镜像的相关概念及工作流程吧。

### 1.1 镜像

镜像（image）是一堆只读层（read-only layer）的统一视角，也许这个定义有些难以理解，下面的这张图能够帮助您理解镜像的定义。

![镜像层结构](https://xcbeyond.cn/blog/containers/dockerfile-best-practices/docker-image-layer.png)

从左边我们看到了多个只读层，它们重叠在一起。除了最下面一层，其它层都会有一个指针指向下一层。这些层是 Docker 内部的实现细节，并且能够在主机的文件系统上访问到。统一文件系统技术能够将不同的层整合成一个文件系统，为这些层提供了一个统一的视角，这样就隐藏了多层的存在，在用户的角度看来，只存在一个文件系统。我们可以在图片的右边看到这个视角的形式。

您可以在您的主机文件系统上找到有关这些层的文件。需要注意的是，在一个运行中的容器内部，这些层是不可见的。在我的主机上，我发现它们存在于 `/var/lib/docker/overlay2` 目录下。

### 1.2 镜像分层结构

为什么说是镜像分层结构，因为 Docker 镜像是以层来组织的，可以通过命令 `docker image inspect <image>` 或者 `docker inspect <image>` 来查看镜像包含哪些层。

例如，镜像 busybox ：

```sh
xcbeyond@xcbeyonddeMacBook-Pro ~ % docker inspect busybox
[
    {
        "Id": "sha256:3c277069c6ae3f3572998e727b973ff7418c3962b9403de4b3a3f8624399b8fa",
        "RepoTags": [
            "busybox:latest"
        ],
        "RepoDigests": [
            "busybox@sha256:d2b53584f580310186df7a2055ce3ff83cc0df6caacf1e3489bff8cf5d0af5d8"
        ],
        "Parent": "",
        "Comment": "",
        "Created": "2022-04-14T00:39:25.923517152Z",
        "Container": "39aaf4eecc48824531078c316f5b16e97549417e07c8f90b26ae16053111ea57",
        "ContainerConfig": {
            "Hostname": "39aaf4eecc48",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/bin/sh",
                "-c",
                "#(nop) ",
                "CMD [\"sh\"]"
            ],
            "Image": "sha256:3289bc85dc0eba79657979661460c7f6f97688ad8a4f93174e0cabdd6b09a365",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {}
        },
        "DockerVersion": "20.10.12",
        "Author": "",
        "Config": {
            "Hostname": "",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "sh"
            ],
            "Image": "sha256:3289bc85dc0eba79657979661460c7f6f97688ad8a4f93174e0cabdd6b09a365",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": null
        },
        "Architecture": "arm64",
        "Variant": "v8",
        "Os": "linux",
        "Size": 1411540,
        "VirtualSize": 1411540,
        "GraphDriver": {
            "Data": {
                "MergedDir": "/var/lib/docker/overlay2/e89181e7cadd3a6ee49f66bae34fed369621a1a5cfbe0003ce4621d0eec020e6/merged",
                "UpperDir": "/var/lib/docker/overlay2/e89181e7cadd3a6ee49f66bae34fed369621a1a5cfbe0003ce4621d0eec020e6/diff",
                "WorkDir": "/var/lib/docker/overlay2/e89181e7cadd3a6ee49f66bae34fed369621a1a5cfbe0003ce4621d0eec020e6/work"
            },
            "Name": "overlay2"
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:31a5597e16d3c5adaaf5826162216e256126d2fbf1beaa2b6c45c1822a2b9ca3"
            ]
        },
        "Metadata": {
            "LastTagTime": "0001-01-01T00:00:00Z"
        }
    }
]
```

其中，RootFS 就是镜像 busybox:latest 的镜像层，只有一层，这层数据是存储在宿主机哪里的呢？动手实践的同学会在上面的输出中看到一个叫做 GraphDriver 的字段内容如下:

```json
"GraphDriver": {
            "Data": {
                "MergedDir": "/var/lib/docker/overlay2/e89181e7cadd3a6ee49f66bae34fed369621a1a5cfbe0003ce4621d0eec020e6/merged",
                "UpperDir": "/var/lib/docker/overlay2/e89181e7cadd3a6ee49f66bae34fed369621a1a5cfbe0003ce4621d0eec020e6/diff",
                "WorkDir": "/var/lib/docker/overlay2/e89181e7cadd3a6ee49f66bae34fed369621a1a5cfbe0003ce4621d0eec020e6/work"
            },
            "Name": "overlay2"
        }
```

GraphDriver 负责镜像本地的管理和存储以及运行中的容器生成镜像等工作，可以将 GraphDriver 理解成镜像管理引擎，我们这里的例子对应的引擎名字是 overlay2（overlay 的优化版本）。除了 overlay 之外，Docker 的 GraphDriver 还支持 btrfs、aufs、devicemapper、vfs 等。

我们可以看到其中的 Data 包含了多个部分，这个对应 OverlayFS 的镜像组织形式，虽然我们上面的例子中的 busybox 镜像只有一层，但是正常情况下很多镜像都是由多层组成的。

### 1.3 Dockerfile、镜像、容器间的关系

Dockerfile 是软件的原材料，Docker 镜像是软件的交付品，而 Docker 容器则可以认为是软件的运行态。从应用软件的角度来看，Dockerfile、Docker 镜像与 Docker 容器分别代表软件的三个不同阶段，Dockerfile 面向开发，Docker 镜像成为交付标准，Docker 容器则涉及部署与运维，三者缺一不可，合力充当 Docker 体系的基石。

简单来讲，Dockerfile 构建出 Docker镜像，通过 Docker 镜像运行Docker容器。

我们可以从 Docker 容器的角度，来反推三者的关系，如下图：

![Docker 镜像结构](https://xcbeyond.cn/blog/containers/dockerfile-best-practices/docker-image-structure.jpeg)

## 2、Dockerfile

Dockerfile 是一个用来构建镜像的文本文件，文本内容包含了一条条构建镜像所需的指令和说明，它是构建镜像的关键。

一个 Docker 镜像包含了很多只读层，每一层都由一个 Dockerfile 指令构成，这些层堆叠在一起，每一层都是前一层变化的增量。例如：

```dockerfile
FROM ubuntu:18.04
COPY . /app
RUN make /app
CMD python /app/app.py
```

每条指令都会创建一层：

* FROM：从 ubuntu:18.04 Docker 镜像创建了一层，也作为基础镜像层。
* COPY：从 Docker 客户端的当前目录添加文件。
* RUN： 执行 make 命令.
* CMD：指定要在容器中运行的命令。

上述就是一个简单的 Dockerfile 文件，再通过 `docker build -t` 命令便可直接构建出镜像。

在这里就不过多介绍 Dockerfile 的各个指令的用法，更多更详细的可参考：[Dockerfile reference](https://docs.docker.com/engine/reference/builder/)

## 3、Dockerfile 的最佳实践

本节将列举出一些最佳实践技巧，来帮助我们更好的写好 Dockerfile。

### 3.1 尽可能使用官方镜像作为基础镜像

Docker 镜像是基于基础镜像构建而来，因此选择的基础镜像越恰当，我们要做的底层工作就越少。比如，如果构建一个 Java 应用镜像，选择一个 openjdk 镜像作为基础比选择一个 alpine 镜像更简单。

尽可能使用当前的官方镜像作为基础镜像，无论是从镜像大小，还是安全性来讲，都是比较可靠的。

下面的一些镜像，可根据使用场景来选择合适的基础镜像：

| 镜像名称 | 大小 | 说明和使用场景 |
| --- | --- | --- |
| [busybox](https://hub.docker.com/_/busybox) | 754.7 KB | 一个超级简化版嵌入式 Linux 系统。临时测试用。|
| [alpine](https://hub.docker.com/_/alpine) | 2.68 MB | 一个面向安全的、轻量级的Linux系统，基于musl libc 和 busybox。主要用于测试，也可用于生产环境。 |
| [centos](https://hub.docker.com/_/centos) | 79.65 MB | 主要用于生产环境，支持CentOS/Red Hat，常用于追求稳定性的企业应用。|
| [ubuntu](https://hub.docker.com/_/ubuntu) | 29.01 MB | 主要用于生产环境，常用于人工智能计算和企业应用。 |
| [debian](https://hub.docker.com/_/debian) | 52.4 MB | 主要用于生产环境。 |
| [openjdk](https://hub.docker.com/_/openjdk) | 161.02 MB | 主要用于  Java 应用。|

### 3.2 减少 Dockerfile 指令的行数

Dockerfile 中每一行指令都代表了一层，多一层都可能带来镜像大小变大。

因此，在实际编写 Dockerfile 时，可以将同类操作放在一起来避免多行指令，更有助于促进层缓存。比如将多条 RUN 操作进行合并，并用 `;\` 或者 `&&` 连接在一起。

（减少指令行数，并不意味着越少越好，需要从改动频繁程度来决定是否合并为一条指令。）

例如下面的 Dockerfile，会执行多条命令，通过 `;\` 连接将其用一条 RUN 指令来完成。

```dockerfile
FROM node:6.14
LABEL MAINTAINER xcbeyond

RUN npm install gitbook-cli -g;\
   gitbook -V; \
   npm install svgexport -g --unsafe-perm

CMD ["/bin/sh"]
```

### 3.3 改动不频繁的内容往前放

对于 Docker 镜像而言，每一层都代表了 Dockerfile 中的一行指令，每一层都是前一层变化的增量。例如一个 Docker 镜像有ABCD 四层，B 层修改了，那么 BCD 都会变化。

因此，在编写 Dockerfile 时，尽量将改动不频繁的内容往前放，即：将系统依赖往前写，因为像 apt, yum 这些安装的东西，是很少修改的。然后写应用的库依赖，比如 pip install，最后 copy 应用,编译应用。

例如下面这个 Dockerfile，就会在每次代码改变的时候都重新 Build 大部分层，即使只改了一个页面的标题。

```dockerfile
FROM python:3.7-buster
 
# copy source
RUN mkdir -p /opt/app
COPY myapp /opt/app/myapp/
WORKDIR /opt/app

# install dependencies nginx
RUN apt-get update && apt-get install nginx
RUN pip install -r requirements.txt
RUN chown -R www-data:www-data /opt/app
 
# start server
EXPOSE 8020
STOPSIGNAL SIGTERM
CMD ["/opt/app/start-server.sh"]
```

我们可以改成，先安装 Nginx，再单独 copy requirements.txt，然后安装 pip 依赖，最后 copy 应用代码。

```dockerfile
FROM python:3.7-buster
 
# install dependencies nginx
RUN apt-get update && apt-get install nginx
COPY myapp/requirements.txt /opt/app/myapp/requirements.txt
RUN pip install -r requirements.txt
 
# copy source
RUN mkdir -p /opt/app
COPY myapp /opt/app/myapp/
WORKDIR /opt/app
 
RUN chown -R www-data:www-data /opt/app
 
# start server
EXPOSE 8020
STOPSIGNAL SIGTERM
CMD ["/opt/app/start-server.sh"]
```

### 3.4 编译和运行需分离

我们在编译应用时很多时候会用到很多编译工具、编译环境，例如：node、Golang 等，但是编译后，运行时却不再需要。这样的编译环境往往占用很大，使得镜像额外变大。

因此，可以将应用事先在某个固定编译环境编译完成，得到编译后的二进制文件，再将其 COPY 到镜像中即可，这样镜像中只包含应用的运行二进制文件。

例如下面这个 Dockerfile，将 Golang 程序编译好的二进制文件 app,构建到镜像中：

```dockerfile
FROM alpine:latest

LABEL maintainer xcbeyond

WORKDIR /app

COPY app /app

CMD ["/app/app"]
```

### 3.5 删除不需要的依赖项

Docker 镜像应该尽可能小。在编写 Dockerfile 时仅包含基本内容，不要引入无关内容，从而使得镜像大小更小、构建速度更快，并且减少受攻击的可能面。

镜像更小，也更利于存放到镜像仓库，减少网络带宽开销。

不要安装应用程序实际不使用的任何包、库。

### 3.6 避免凭证构建到镜像

这是最常见和最危险的 Dockerfile 问题之一。在构建镜像过程中，复制配置文件可能很诱人，但你切记可能会引入很大的安全隐患。

在 Dockerfile 中通过 COPY 指令将任何配置文件内容都复制到你的镜像，并且任何可以访问它的人都可以访问它。如果这个配置文件中，无意间包含了数据库密码配置，那么你就彻底将这些密码暴露给了所有使用该镜像的所有人。

为了避免这类问题，必须将配置密钥、敏感数据只能提供给具体的容器，而不是提供给构建它们的镜像。可使用环境变量、挂载卷等方式在容器启动时注入数据。这样就避免了意外的信息暴露，并确保你的镜像可跨环境重复使用。

---

参考文章：

1. [如何选择Docker基础镜像](https://blog.csdn.net/nklinsirui/article/details/80967677)
2. [Top 20 Dockerfile best practices](https://sysdig.com/blog/dockerfile-best-practices/)
3. [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
