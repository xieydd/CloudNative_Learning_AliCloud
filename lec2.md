### Lec2: 容器基本概念

##### 1. 什么是容器？

- 物理机上执行 ps 看到的进程之间可以相互通信，相互之间共享文件系统，使用相同的系统资源。

  - 高级别进程可以攻击低级别进程
  - 进程依赖冲突
  - 共享文件会冲突
  - 资源抢占问题

- 三大隔离

  - chroot 可以让进程拥有自己独立的文件系统
  - namespace 对资源视图进行隔离
  - 使用 cgroup 对于系统资源进行限制

  拥有独立的文件系统，视图隔离（能看见部分进程、拥有独立的主机名等等），资源可限制的进程集合叫做容器。

##### 2. 什么是镜像？

- 镜像就是容器的所有依赖环境和文件的集合。

- Dockerfile 构建镜像 

  -  其中的一些指令 ADD 、CMD等相当于语法糖。
  - 构建时会产生文件系统变化(changeset)，类似于disk snapshot。
  - 文件复用可以提高分发效率，减少磁盘的压力。

  ```dockerfile
  FROM golang:1.12.1-alpine
  
  # setting PWD to /go/src/app/
  WORKDIR /go/src/app
  # copy current files to /go/src/app
  COPY . .
  #get all dependecy
  RUN go get -d -v ./..
  #build the application and install it
  RUN go install -v ./..
  CMD ["app"]
  ```

- 封装，提交，运行镜像, Build Once, Run Anywhere

  ```shell
  docker build . -t app:v1
  docker push app:v1
  # other machine
  docker pull app:v1
  docker run -it (-d) app:v1 --name app1 /bin/bash
  ```

##### 3. 容器的什么周期

- 单进程模型：Init 进程 = 容器生命周期， exec 进入操作也属于 Init 进程

  - 问题：有状态应用，需要持久化数据；

- 数据卷： 生命周期独立于容器

  ```shell
  # 1. 本地绑定
  docker run /tmp:/tmp busybox:1.25 sh -c "date > /tmp/time.log"
  cat /tmp/time.log
  # 2. 容器引擎管理
  docker create volume demo
  docker run demo:/tmp busybox:1.25 sh -c "date > /tmp/time.log"
  docker run demo:/tmp busybox:1.25 sh -c "cat /tmp/time.log"
  ```

##### 4. Moby (原docker) 容器引擎架构

- Moby Daemon 依赖 containerd
- Containerd 是容器运行时管理引擎， 包含 containerd-shim 如 runC、gVisor、kata containerd 用于管理容器的生命周期；插件化管理

##### 5. 容器和 VM

- VM 依赖于 Hypervisor 技术：Guest OS 拥有独立的内核、隔离效果更好、资源损耗大
- 容器：无 Guest OS ，进程级隔离，启动快，但是隔离的资源较少；gVisor 和 kata container 提供强隔离保证安全性

