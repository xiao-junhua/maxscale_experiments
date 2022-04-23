# Docker 学习
## 安装Docker

1. 在自身机器上安装docker

参考****[How To Install and Use Docker on Ubuntu 20.04](https://phoenixnap.com/kb/install-docker-on-ubuntu-20-04#:~:text=1%20Updating%20the%20Local%20Repository%202%20Uninstalling%20Old,output%20should%20verify%20Docker%20is%20active%20%28running%29.%20)****

## Docker学习

Docker是容器，相当于独立在在操作系统上的一个库

> Docker系统有两个程序：**docker服务端和docker客户端**。其中docker服务端是一个服务进程，管理着所有的容器。docker客户端则扮演着docker服务端的远程控制器，可以用来控制docker的服务端进程。大部分情况下，docker服务端和客户端运行在一台机器上。
> 
1. 镜像相关
- docker search 镜像名字
- docker pull learn/image_name
1. 容器运行程序

> docker容器可以理解为在沙盒中运行的进程。这个沙盒包含了该进程运行所必须的资源，包括文件系统、系统类库、shell 环境等等。但这个沙盒默认是不会运行任何程序的。你需要在沙盒中运行一个进程来启动某一个容器。这个进程是该容器的唯一进程，所以当该进程结束的时候，容器也会完全的停止。
> 

**docker run**命令有两个参数，一个是镜像名，一个是要在镜像中运行的命令。

1. **Dockfile** — 用于创建docker image的一个脚本

```docker
# 从node:12-alpine下载镜像
FROM node:12-alpine
# Adding build tools to make yarn install work on Apple silicon / arm64 machines
RUN apk add --no-cache python2 g++ make
WORKDIR /app
COPY . .
RUN yarn install --production

# 当起一个docker镜像的时候，默认命令
CMD ["node", "src/index.js"]
```

使用的命令是 docker build -t getting-started .  — 新建一个名字是getting-started的镜像

1. 如何管理镜像
- 修改app后想要更新镜像

```bash
$ docker build -t getting-started .

# 查找到旧的镜像
$ docker ps

# 删除旧的镜像
$ # Swap out <the-container-id> with the ID from docker ps
docker stop <the-container-id>
$ docker rm <the-container-id>

# 启动新的镜像
```

1. 上传镜像到docker hub

```bash
# 给镜像打一个标签 -- 对应repo的名字
$ docker tag getting-started YOUR-USER-NAME/getting-started
```

1. 持久化

- **Named Volume**

container之间是互相独立的，所以你每次修改的东西，如果把container停止之后，都不会保存，

要想保存修改内容，可以使用sqlite保存，具体命令如下

```bash
# 新建一块磁盘
$ docker volume create todo-db
# 标注存储区域
$ docker run -dp 3000:3000 -v todo-db:/etc/todos getting-started

# 查看实际存储位置
$ docker volume inspect <todo-db>
```

- **Bind Mounts**

将host os里面的文件链接到container之中

```bash
docker run -dp 3000:3000 \
    -w /app -v "$(pwd):/app" \
    node:12-alpine \
    sh -c "yarn install && yarn run dev"
# -w container中设置working directory
# -v "$(pwd):/app" bind mount到container之中
```

1. 多个container之间交互

以一个container存到MySQL中为例

- 创建网络

```bash
$ docker network create todo-app
```

- 新起一个mysql的容器进程，并把它添加到网络中

```bash
$ docker run -d \
    --network todo-app --network-alias mysql \
    -v todo-mysql-data:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=secret \
    -e MYSQL_DATABASE=todos \
    mysql:5.7
# --network <network_name>表示将容器进程添加到该网络之中
# --network-alias mysql表示设置mysql所在的host别名
# -e 设置mysql的环境变量
# mysql:5.7表示container image
# -v 指定存储区域 <volume_name>:<container_part>, 这是是默认创建了一个volume

# 进一步地，你可以运行mysql容器里面的mysql
$ docker exec -it <mysql-container-id> mysql -p
```

- [optional] 使用 [nicolaka/netshoot](https://github.com/nicolaka/netshoot) container来查看对应container网络中的host

```bash
$ docker run -it --network todo-app nicolaka/netshoot
$ dig mysql
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2fb510df-b8c2-4910-9eed-f31c1dbac801/Untitled.png)

- 添加新的进程到container网络之中用于通讯

```bash
$ docker run -dp 3000:3000 \
  -w /app -v "$(pwd):/app" \
  --network todo-app \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=secret \
  -e MYSQL_DB=todos \
  node:12-alpine \
  sh -c "yarn install && yarn run dev"
```

这样就可以确保在app的container进程中做的修改能够保存到mysql里面

1. 采用docker-compose创建multi-container进程

将命令改为docker-compose.yml

```bash
version: "3.8"

services:
  app:
    image: node:12-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos
  
  mysql:
    image: mysql:5.7
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment: 
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos
    
volumes:
  todo-mysql-data:
```