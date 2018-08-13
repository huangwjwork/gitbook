# docker

[TOC]

## 镜像管理

### 镜像常规管理命令

#### 查看本地镜像

```shell
➜  ~ docker images 
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              5699ececb21c        2 days ago          109MB
ubuntu              latest              113a43faa138        3 weeks ago         81.2MB
➜  ~ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              5699ececb21c        2 days ago          109MB
ubuntu              latest              113a43faa138        3 weeks ago         81.2MB

```

#### 查看虚悬镜像

没有仓库名没有tag的镜像，就是虚悬镜像，可以用

```shell
docker image ls -f dangling=true➜  ~ docker image ls -f dangling=true
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```

删除虚悬镜像

```shell
➜  ~ docker image prune
WARNING! This will remove all dangling images.
Are you sure you want to continue? [y/N] y
Total reclaimed space: 0B
```

#### 查看实际磁盘占用

```shell
➜  ~ ➜  ~ docker system df 
TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              2                   1                   190.1MB             109MB (57%)
Containers          1                   1                   0B                  0B
Local Volumes       0                   0                   0B                  0B
Build Cache
```

#### 中间层镜像

被顶层镜像依赖的镜像叫做中间层镜像，没有tag

```shell
➜  ~ sudo docker image ls -a
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              5699ececb21c        2 days ago          109MB
ubuntu              16.04               5e8b97a2a082        3 weeks ago         114MB
ubuntu              latest              113a43faa138        3 weeks ago         81.2MB

```

#### 镜像摘要

```shell
➜  ~ docker image ls --digests 
REPOSITORY          TAG                 DIGEST                                                                    IMAGE ID            CREATED             SIZE
nginx               latest              sha256:62a095e5da5f977b9f830adaf64d604c614024bf239d21068e4ca826d0d629a4   5699ececb21c        2 days ago          109MB
ubuntu              16.04               sha256:b050c1822d37a4463c01ceda24d0fc4c679b0dd3c43e742730e2884d3c582e3a   5e8b97a2a082        3 weeks ago         114MB
ubuntu              latest              sha256:778d2aed25eb85ec40548addb582cbffd0ca6fe5d2a382cb7f3a8958c1ed50d6   113a43faa138        3 weeks ago         81.2MB

```

#### 删除镜像

```shell
docker image rm image_ID
```

### 创建镜像的多种方法

#### commit

对原有docker image内容进行修改，然后commit为一个新的镜像，这样会对镜像内很多文件进行修改，实际生产环境不推荐使用commit

* pull一个nginx

  ```shell
  ➜  ~ sudo docker run --name webserver -d -p 80:80 nginx
  ```

* 连接nginx

  ```shell
  ➜  ~ docker exec -it webserver bash 
  ```

* 修改html

  ```shell
  ➜  ~ x.htmlc0e9a0ede4a:/# echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index
  ```

* 查看镜像的改动

  ```shell
  ➜  ~ sudo docker diff webserver
  C /root
  A /root/.bash_history
  C /run
  A /run/nginx.pid
  C /usr/share/nginx/html/index.html
  C /var/cache/nginx
  D /var/cache/nginx/client_temp
  D /var/cache/nginx/fastcgi_temp
  D /var/cache/nginx/proxy_temp
  D /var/cache/nginx/scgi_temp
  D /var/cache/nginx/uwsgi_temp
  ➜  ~ 
  ➜  ~ x.htmlc0e9a0ede4a:/# echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index
  
  ```

* commit镜像

  ```shell
  docker commit \
  --author "Tao Wang <twang2218@gmail.com>" \
  --message "修改了默认网页" \
  webserver \
  nginx:v2
  ```

* 查看镜像操作历史

  ```shell
  ➜  ~ sudo docker history nginx:v2
  [sudo] wyny 的密码： 
  IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
  0f7f4f5291b4        27 minutes ago      nginx -g daemon off;                            97B                 修改了默认网页
  5699ececb21c        5 days ago          /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemon…   0B                  
  <missing>           5 days ago          /bin/sh -c #(nop)  STOPSIGNAL [SIGTERM]         0B                  
  <missing>           5 days ago          /bin/sh -c #(nop)  EXPOSE 80/tcp                0B                  
  <missing>           5 days ago          /bin/sh -c ln -sf /dev/stdout /var/log/nginx…   22B                 
  <missing>           5 days ago          /bin/sh -c set -x  && apt-get update  && apt…   53.7MB              
  <missing>           5 days ago          /bin/sh -c #(nop)  ENV NJS_VERSION=1.15.0.0.…   0B                  
  <missing>           5 days ago          /bin/sh -c #(nop)  ENV NGINX_VERSION=1.15.0-…   0B                  
  <missing>           5 days ago          /bin/sh -c #(nop)  LABEL maintainer=NGINX Do…   0B                  
  <missing>           5 days ago          /bin/sh -c #(nop)  CMD ["bash"]                 0B                  
  <missing>           5 days ago          /bin/sh -c #(nop) ADD file:28fbc9fd012eef727…   55.3MB        
  ```

  

#### dockerfile

```shell
➜  docker cat nginx_v1/Dockerfile 
from nginx
run echo '<h1>Hello,Docker!</h1>'  >  /usr/share/nginx/html/index.html

```

##### 构建镜像

**注意：**build最后一位的参数.指的并不是dockerfile的目录，而是context的目录

```shell
docker build -t nginx:v3 .
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM nginx
---> e43d811ce2f4
Step 2 : RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
---> Running in 9cdc27646c7b
---> 44aa4490ce2c
Removing intermediate container 9cdc27646c7b
Successfully built 44aa4490ce2c
```

##### context目录

docker镜像构筑原理：client通过与docker服务提供的restAPI通信，由server完成构筑，因此build中的路径.是context目录

docker从本地COPY内容至镜像时，格式为`COPY ./package.json  /app/`，将本地的package.json拷贝到镜像的/app下，然后这里的./package.json并不是当前目录，而是build时写入的context目录

一般为了防止出现错误，建议新建一个镜像文件夹，写入dockerfile，并将需要COPY的文件放在镜像文件夹下，在镜像文件夹目录下执行build命令

#### git repo

基于仓库内的11.0目录下的dockerfile

```shell
➜  ~ sudo docker build https://github.com/twang2218/gitlab-ce-zh.git\#:11.0
[sudo] wyny 的密码： 
Sending build context to Docker daemon  5.632kB
Step 1/23 : FROM gitlab/gitlab-ce:11.0.1-ce.0 as builder
11.0.1-ce.0: Pulling from gitlab/gitlab-ce
b234f539f7a1: Already exists 
55172d420b43: Already exists 
5ba5bbeb6b91: Already exists 
43ae2841ad7a: Already exists 
f6c9c6de4190: Already exists 
e6597e975df6: Downloading [=====>                                             ]  3.006MB/26.24MB
73dc92c66025: Download complete 
5282194f87d6: Download complete 
df88dfe59bb1: Download complete 
dffab5bd4f6d: Download complete 
945907026b4f: Downloading [>                                                  ]   3.26MB/443.6MB
```

#### tar

```shell
➜  ~ docker build http://server/context.tar.gz
```

#### std_input

这种方式不存在build的context目录，所以不能COPY文件

```shell
➜  ~ docker build - < Dockerfile
```

```shell
➜  ~ cat Dockerfile | docker build -
```

### dockerfile详解

#### FROM

以一个镜像为基础在其基础上进行定制，可以基础镜像、服务镜像，还有特殊镜像scratch，表示并不存在的特殊镜像

scratch内没有任何内容，没有操作系统，可以直接复制文件进镜像后执行，这样制作出来的镜像体积很小，很适合GO语言开发的应用

#### RUN

每一个RUN都对应一次commit，有多少RUN就会在docker上封装几层

- ​	shell命令
- exec文件



**多次RUN**

为了编译安装redis写了7条run，镜像会封装7次，最终的镜像会很臃肿

```dockerfile
FROM debian:jessie
RUN apt-get update
RUN apt-get install -y gcc libc6-dev make
RUN wget -O redis.tar.gz "http://download.redis.io/releases/redis-3.2.5.tar.gz"
RUN mkdir -p /usr/src/redis
RUN tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1
RUN make -C /usr/src/redis
RUN make -C /usr/src/redis install
```

**一次RUN**

一样的效果只有一次run，并且在封装后删除了不必要的改动，一次commit，最后形成的docker镜像会很轻量

```dockerfile
FROM debian:jessie
RUN buildDeps='gcc libc6-dev make' \
&& apt-get update \
&& apt-get install -y $buildDeps \
&& wget -O redis.tar.gz "http://download.redis.io/releases/redis-3.2.5.tar.gz" \
&& mkdir -p /usr/src/redis \
&& tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
&& make -C /usr/src/redis \
&& make -C /usr/src/redis install \
&& rm -rf /var/lib/apt/lists/* \
&& rm redis.tar.gz \
&& rm -r /usr/src/redis \
&& apt-get purge -y --auto-remove $buildDeps
```

#### COPY 

```dockerfile
COPY [PASS1,PASS2,PASS3] DES
```

* 目标路径不需要提前创建
* 源文件的各种属性都会得到保留

#### ADD

高级复制

* 源路径为URL，可以直接下载文件到指定目录
* 源路径为压缩包（gzip，bzip2，bzip2），可以直接解压缩到目标路径

在选择COPY和ADD遵循的原则，文件复制使用COPY，需要自动解压缩时使用ADD

#### CMD

- shell，会被转变为sh -c
- exec ，CMD ["可执行文件","参数1",“参数2”]

用于指定默认的容器主进程的启动命令，会被docker run 后面的命令所覆盖

* ​			当docker run不指定运行的命令时，则执行CMD的命令
* 当docker run制定了运行的命令时，则执行指定的命令

**注意：docker不能有后台的概念，所有的应用都应在前台执行**

* 错误的写法：`CMD  service nginx start`

  这样做对于docker来讲执行service nginx start后这条命令就已经结束了，容器主进程执行完成，失去存在意义，就会退出容器

* 正确的写法：`CMD ["nginx","-g"],"daemon off;"`

#### ENTRYPOINT

- shell，会被转变为sh -c
- exec ，ENTRYPOINT ["可执行文件","参数1",“参数2”]

docker run 后面的参数会传递给ENTERPOINT作为参数

dockerfile

```dockerfile
FROM ubuntu:latest
ENTRYPOINT ["netstat"]
```

docker run，tunlp会被传递给entrypoint，最后运行的是netstat -tunlp

```shell
docker run -it ubuntu -tunlp 
```

#### ENV

设置环境变量，无论是RUN还是docker运行时的应用都可以调用，调用方式与shell一致

* ENV <key> <value>

* ENV <key1>=<value1> <key2>=<value2>

  ```dockerfile
  ENV NODE_VERSION 7.2.0
  RUN curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-x64.ta
  r.xz" \
  && curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/SHASUMS256.txt.asc" \
  && gpg --batch --decrypt --output SHASUMS256.txt SHASUMS256.txt.asc \
  && grep " node-v$NODE_VERSION-linux-x64.tar.xz\$" SHASUMS256.txt | sha256sum -c - \
  && tar -xJf "node-v$NODE_VERSION-linux-x64.tar.xz" -C /usr/local --strip-components=
  1 \
  && rm "node-v$NODE_VERSION-linux-x64.tar.xz" SHASUMS256.txt.asc SHASUMS256.txt \
  && ln -s /usr/local/bin/node /usr/local/bin/nodejs
  ```

#### ARG

- ARG <key> <value>
- ARG <key1>=<value1> <key2>=<value2>

定义默认环境变量，会被docker build中的 --build-arg <参数名>=<值>

与ENV类似，区别在于ARG只是构建环境的环境变量，容器运行时不存在

#### VOLUME

定义默认的匿名卷，容器应该保持容器存储层不发生写操作，需要保存动态数据时应该保存在卷volume中，在dockerfile中定义匿名卷，防止运行时用户忘记挂载卷，会被docker run -v 覆盖

```dockerfile
VOLUME /data
```

```shell
docker run -d -v mydata:/data xxxx
```

#### EXPOSE

申明端口，并非打开端口，与docker run -p有区别

#### WORKDIR

指定工作目录，不存在时，workdir会建立该目录

RUN cd /app只是改变了当前的层的工作目录，当运行第二个RUN时，并不在cd后的目录，还是回到workdir

#### UESR

指定RUN，CMD，ENTRYPOINT的用户

```dockerfile
RUN groupadd -r redis && useradd -r -g redis redis
USER redis
RUN [ "redis-server" ]
```

gosu

```dockerfile
RUN groupadd -r redis && useradd -r -g redis redis
# 下载 gosu
RUN wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/1.7/
gosu-amd64" \
&& chmod +x /usr/local/bin/gosu \
&& gosu nobody true
# 设置 CMD,并以另外的用户执行
CMD [ "exec", "gosu", "redis", "redis-server" ]
```

#### HEALTHCHECK

检查容器状态，healthy | unhealthy

* --interval=<> ，健康检查的时间间隔
* --timeout=<>，健康检查的超时时间
* --retries=<>，连续失败的次数

```dockerfile
FROM nginx
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
HEALTHCHECK --interval=5s --timeout=3s \
CMD curl -fs http://localhost/ || exit 1
```

#### ONBUILD

当前dockerfile build时不会执行，只有在以build后的镜像为基础镜像build时才会执行



### 多阶段构建 ？？？

### 其他制作镜像的方式

#### rootfs导入

```shell
docker import \
http://download.openvz.org/template/precreated/ubuntu-14.04-x86_64-minimal.tar.gz\
openvz/ubuntu:14.04
```

下载镜像并导入为openvz/ubuntu:14.04

#### docker save & docker load

从一台server save并在另一台server laod

```shell
docker save  alpine |  gzip > alpine-latest.tar.gz
```

```shell
docker laod -i alpine-latest.tar.gz
```

## 容器操作

### 启停

docker run启动过程

1. 检查是否存在指定的镜像，不存在则去仓库下载
2. 利用镜像创建并启动一个容器
3. 分配文件系统，在制度的镜像层外面股在一层可读写层
4. 从宿主机配置的王巧借口桥接一个虚拟接口到容器
5. 从地址池配置一个IP地址
6. 执行用户指定的应用程序
7. 执行完毕后容器终止

* 基于镜像启动容器

  ```shell
  ➜  ~ docker run ubuntu /bin/echo 'hello world' 
  hello world
  ➜  ~ docker ps -a 
  41a98623a795        ubuntu              "/bin/echo 'hello wo…"   6 minutes ago       Exited (0) 6 minutes ago                                            clever_bell
  
  ```

* 启动已终止的容器

  ```shell
  docker start container_ID 
  docker container start container_ID
  ```

* 查看已启动的容器

  ```shell
  docker container ls
  docker ps 
  ```

* 查看所有容器

  ```shell
  docker container ls -a 
  docker ps -a 
  ```

### 守护状态

```shell
docker run  -d  --name nginx-daemon nginx:latest 
```

### 进入容器

#### run

* -i 开启std_input
* -t 开启tty

```shell
docker run -it --name ubuntu:latest /bin/➜  ~ docker run -it --name bash_ubuntu  ubuntu:latest /bin/bash
```

#### attach

进入到容器界面

* 当容器run运行bash时，attach会进入bash
* 当容器run运行其他服务，进入服务界面

#### exec

docker exec -i -t  container_ID CMD

```shell
➜  ~ docker exec➜  ~ docker exec -it 3d9 /bin/bash 
root@3d94524412ec:/# 
```

### 导入导出容器

export

```shell
➜  ~ docker container ls 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
3d94524412ec        ubuntu              "/bin/bash"         2 minutes ago       Up About a minute                       jovial_fermat
➜  ~ docker export 3d94 > ubuntu_export.tar 
➜  ~ ls -lh
-rw-r--r-- 1 wyny wyny  69M 7月   4 15:09 ubuntu_export.tar


```

import

```shell
➜  ~ cat ubuntu_export.tar |  docker import - test/ubuntu:v1
sha256:a48224e5ba2a59d2409d7fecca9b05d574e4962dfbf8144a66d53b1500d256f1
➜  ~ docker image ls 
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
test/ubuntu         v1                  a48224e5ba2a        6 seconds ago       69.8MB
```

### 删除

```shell
➜  ~ docker container ls -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
7ca98669632a        ubuntu              "/bin/bash"         9 seconds ago       Exited (0) 7 seconds ago                       condescending_stonebraker
➜  ~ docker container rm 7c
7c
```

清理所有stop的container

```shell
➜  ~ docker container ls -a 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                          PORTS               NAMES
f1491993f9a8        ubuntu              "/bin/bash"         8 seconds ago       Exited (0) 6 seconds ago                            happy_hopper
2a723dd1559b        ubuntu              "/bin/bash"         15 seconds ago      Exited (0) 13 seconds ago                           romantic_curran
3d94524412ec        ubuntu              "/bin/bash"         7 minutes ago       Exited (0) About a minute ago                       jovial_fermat
➜  ~ docker container prune 
WARNING! This will remove all stopped containers.
Are you sure you want to continue? [y/N] y
Deleted Containers:
f1491993f9a878ded0eb17bc856c180a5b09f2a6037e279a03d4de884f49ff44
2a723dd1559b8e3f6448b5d2dc17a4322f250bc3c562c39034dfdf7851bc3e55
3d94524412ecdb32071a1e03d8bc86c4519fd868472e414fd9b3174300ea379d

Total reclaimed space: 19B
➜  ~ docker container ls -a 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
➜  ~ 

```

## docker数据管理

### 数据卷

可供一个或多个容器使用的特殊目录，绕过UFS（union FS统一文件系统），具备如下特性：

* 可以在容器之间共享和重用

* 修改立刻生效
* 更新数据库不影响容器
* 默认一直存在，即使容器被删除

#### 创建数据卷

```shell
➜  ~ docker volume create my-vol
my-vol
```

#### 查看所有数据卷

```shell
➜  ~ docker volume ls 
DRIVER              VOLUME NAME
local               042c76be19978e725c59d0265aad78f972666807b2b4ae2eaf2fda1628e149c7
local               0f14d341dcd6ac228c437eb644ecaad595265f90e2daa1f39ecdf1697f778640
local               3383d181e3fe032f4ddbcf7256cd4a9b6b8febedbc4c00dfadd72e71a3982b78
local               3c7c0b5c6c24f43be58d1a3b1b71cb71af274c1344c718d38a55592ed84b6d99
local               426a573ca27fca528c4f72a6edca4d8df3159cfe81fd99c2d5025f872e43efcd
local               49f3d6308563f992ba413946cb996fc52081867c091b0fd3d92873264e7f26d6
local               4d936140426aeee859d7f6c87067abe31f37e1360e8dc83ff34bcc4b56c51ed0
local               703a5a7fef05f6256622c889796d967b828c7c8fd1e1c2f36a3887945fffc6c2
local               7cf48886a15d45378705f854b50726cbaabf3f92d2a46f6f674182bd95102c33
local               9abe590569342b9f05c41eafe1d071648fecf69f084750270c1bbb8923953cbf
local               a24668e656ed8a43b8e56c7b9168746185cacbed1f6c3af5ed37d9e17418044e
local               b633294eb0f48fbc47a00990696853ae25b21c565702d3848a004efd4cc76d1a
local               b74a4636deabb951b966c9636709d5de7db87c255666f28a5a772df66e9afc87
local               c1670fee249053af9295f6a371c78bd5af372de01257a6dc3c77a3cc9997a848
local               f46caff117a4b2b9c179ce569f3880d596ed0c2500b342bd809fe92382a956de
local               my-vol
```

#### 查看数据卷信息

```shell
➜  ~ docker volume inspect my-vol
[
    {
        "CreatedAt": "2018-07-04T16:33:50+08:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-vol/_data",
        "Name": "my-vol",
        "Options": {},
        "Scope": "local"
    }
]

```

#### 删除数据卷

```shell
➜  ~ docker volume rm my-vol
my-vol
➜  ~ docker volume ls
DRIVER              VOLUME NAME
local               042c76be19978e725c59d0265aad78f972666807b2b4ae2eaf2fda1628e149c7
local               0f14d341dcd6ac228c437eb644ecaad595265f90e2daa1f39ecdf1697f778640
local               3383d181e3fe032f4ddbcf7256cd4a9b6b8febedbc4c00dfadd72e71a3982b78
local               3c7c0b5c6c24f43be58d1a3b1b71cb71af274c1344c718d38a55592ed84b6d99
local               426a573ca27fca528c4f72a6edca4d8df3159cfe81fd99c2d5025f872e43efcd
local               49f3d6308563f992ba413946cb996fc52081867c091b0fd3d92873264e7f26d6
local               4d936140426aeee859d7f6c87067abe31f37e1360e8dc83ff34bcc4b56c51ed0
local               703a5a7fef05f6256622c889796d967b828c7c8fd1e1c2f36a3887945fffc6c2
local               7cf48886a15d45378705f854b50726cbaabf3f92d2a46f6f674182bd95102c33
local               9abe590569342b9f05c41eafe1d071648fecf69f084750270c1bbb8923953cbf
local               a24668e656ed8a43b8e56c7b9168746185cacbed1f6c3af5ed37d9e17418044e
local               b633294eb0f48fbc47a00990696853ae25b21c565702d3848a004efd4cc76d1a
local               b74a4636deabb951b966c9636709d5de7db87c255666f28a5a772df66e9afc87
local               c1670fee249053af9295f6a371c78bd5af372de01257a6dc3c77a3cc9997a848
local               f46caff117a4b2b9c179ce569f3880d596ed0c2500b342bd809fe92382a956de

```

#### 删除无主数据卷

数据卷在容器被删除后依然存在，可以删除无助数据卷节约空间

```shell
➜  ~ docker volume ls
DRIVER              VOLUME NAME
local               042c76be19978e725c59d0265aad78f972666807b2b4ae2eaf2fda1628e149c7
local               0f14d341dcd6ac228c437eb644ecaad595265f90e2daa1f39ecdf1697f778640
local               3383d181e3fe032f4ddbcf7256cd4a9b6b8febedbc4c00dfadd72e71a3982b78
local               3c7c0b5c6c24f43be58d1a3b1b71cb71af274c1344c718d38a55592ed84b6d99
local               426a573ca27fca528c4f72a6edca4d8df3159cfe81fd99c2d5025f872e43efcd
local               49f3d6308563f992ba413946cb996fc52081867c091b0fd3d92873264e7f26d6
local               4d936140426aeee859d7f6c87067abe31f37e1360e8dc83ff34bcc4b56c51ed0
local               703a5a7fef05f6256622c889796d967b828c7c8fd1e1c2f36a3887945fffc6c2
local               7cf48886a15d45378705f854b50726cbaabf3f92d2a46f6f674182bd95102c33
local               9abe590569342b9f05c41eafe1d071648fecf69f084750270c1bbb8923953cbf
local               a24668e656ed8a43b8e56c7b9168746185cacbed1f6c3af5ed37d9e17418044e
local               b633294eb0f48fbc47a00990696853ae25b21c565702d3848a004efd4cc76d1a
local               b74a4636deabb951b966c9636709d5de7db87c255666f28a5a772df66e9afc87
local               c1670fee249053af9295f6a371c78bd5af372de01257a6dc3c77a3cc9997a848
local               f46caff117a4b2b9c179ce569f3880d596ed0c2500b342bd809fe92382a956de
➜  ~ docker volume prune
WARNING! This will remove all local volumes not used by at least one container.
Are you sure you want to continue? [y/N] y
Deleted Volumes:
b633294eb0f48fbc47a00990696853ae25b21c565702d3848a004efd4cc76d1a
b74a4636deabb951b966c9636709d5de7db87c255666f28a5a772df66e9afc87
a24668e656ed8a43b8e56c7b9168746185cacbed1f6c3af5ed37d9e17418044e
c1670fee249053af9295f6a371c78bd5af372de01257a6dc3c77a3cc9997a848
703a5a7fef05f6256622c889796d967b828c7c8fd1e1c2f36a3887945fffc6c2
f46caff117a4b2b9c179ce569f3880d596ed0c2500b342bd809fe92382a956de
0f14d341dcd6ac228c437eb644ecaad595265f90e2daa1f39ecdf1697f778640
426a573ca27fca528c4f72a6edca4d8df3159cfe81fd99c2d5025f872e43efcd
3c7c0b5c6c24f43be58d1a3b1b71cb71af274c1344c718d38a55592ed84b6d99
49f3d6308563f992ba413946cb996fc52081867c091b0fd3d92873264e7f26d6
7cf48886a15d45378705f854b50726cbaabf3f92d2a46f6f674182bd95102c33
042c76be19978e725c59d0265aad78f972666807b2b4ae2eaf2fda1628e149c7
4d936140426aeee859d7f6c87067abe31f37e1360e8dc83ff34bcc4b56c51ed0
3383d181e3fe032f4ddbcf7256cd4a9b6b8febedbc4c00dfadd72e71a3982b78
9abe590569342b9f05c41eafe1d071648fecf69f084750270c1bbb8923953cbf

Total reclaimed space: 255.8MB
➜  ~ docker volume ls       
DRIVER              VOLUME NAME

```

#### 启动一个挂在数据卷的容器

```shell
➜  ~ docker volume create vol1
vol1
# 启动容器
➜  ~ docker run -d -p 80:80 --mount  source=vol1,target=/webapp  --name webserver nginx:latest
01b0dad5508c78670c233a64fcb5f83cd5ae6383e37e3f508e668860dd0873ac
➜  ~ docker exec -it webserver bash 
root@01b0dad5508c:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          38G   19G   18G  52% /
tmpfs            64M     0   64M   0% /dev
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda4        38G   19G   18G  52% /webapp
shm              64M     0   64M   0% /dev/shm
tmpfs           3.9G     0  3.9G   0% /proc/scsi
tmpfs           3.9G     0  3.9G   0% /sys/firmware

```

### 挂载主机目录

```shell
➜  nginx_v1  docker run -d -it --name  hostdir  -p 81:80 --mount type=bind,source=/home/wyny/Documents/docker/nginx_v1,target=/nginx_v1  nginx 
01f286e98411befd047b7e4fb5ec1c0b7694a5892d730f1e49b9c55e058d0d2c
➜  nginx_v1 
➜  nginx_v1 
➜  nginx_v1 
➜  nginx_v1 docker exec -it hostdir bash 
root@01f286e98411:/# 
root@01f286e98411:/# 
root@01f286e98411:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          38G   19G   18G  52% /
tmpfs            64M     0   64M   0% /dev
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda4        38G   19G   18G  52% /nginx_v1
shm              64M     0   64M   0% /dev/shm
tmpfs           3.9G     0  3.9G   0% /proc/scsi
tmpfs           3.9G     0  3.9G   0% /sys/firmware
```

#### 查看挂载信息

```shell
docker inspect container_ID
```

```json
        "Mounts": [
            {
                "Type": "bind",
                "Source": "/home/wyny/Documents/docker/nginx_v1",
                "Destination": "/nginx_v1",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            }
        ]
```

#### 挂载单个文件

```shell
docker run --rm -it \
# -v $HOME/.bash_history:/root/.bash_history \
--mount type=bind,source=$HOME/.bash_history,target=/root/.bash_history \
ubuntu:17.10 \
bash
```

## docker网络

### 端口映射

#### 映射主机所有地址的随机端口

docker -P

```shell
➜  .ssh docker run -d  -P  --name nginx_P nginx 
7f4a2442b2c52656d6e4709142f81edefaa075896d036165880f169f4dfbcd63
➜  .ssh docker container ls 
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
7f4a2442b2c5        nginx               "nginx -g 'daemon of…"   7 seconds ago       Up 5 seconds        0.0.0.0:32768->80/tcp   nginx_P

```

#### 映射指定宿主机IP指定端口

```shell
➜  .ssh docker run -d -p 127.0.0.1:80:80 --name nginx80 nginx

7f2277c62fa1aae037555555cb38a75f77bc9f93d2ed82a7db70d115a59ed902
➜  .ssh docker container ls 
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
7f2277c62fa1        nginx               "nginx -g 'daemon of…"   7 seconds ago       Up 4 seconds        127.0.0.1:80->80/tcp   nginx80
```

#### 映射宿主机所有IP的指定端口

```shell
➜  .ssh docker run -d -p 81:80 --name nginx81 nginx
693be8dfb0eca93c5193a419250e116d2188e6c38c54cd0243baba2983c4cf5e
➜  .ssh docker container ls 
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                     NAMES
693be8dfb0ec        nginx               "nginx -g 'daemon of…"   40 seconds ago       Up 38 seconds       0.0.0.0:81->80/tcp        nginx81
```

#### 映射宿主机指定IP的随机端口

```shell
➜  .ssh docker run -d -p 127.0.0.1::80 --name nginx_random nginx 
5655972dbb2ab744b15f65e294164685f37035b71a4acf388c077a54c54f39bf
➜  .ssh docker container ls 
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                     NAMES
5655972dbb2a        nginx               "nginx -g 'daemon of…"   5 seconds ago        Up 3 seconds        127.0.0.1:32768->80/tcp   nginx_random
```

#### 映射多个端口

```shell
➜  .ssh docker run -d -p 22:220  -p 80:80  --name nginx_22_80  nginx
e7223e2e3d2ac784252048752f29884b501f39453ac73e6ae816785393e519ca
➜  .ssh docker container ls 
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                     NAMES
e7223e2e3d2a        nginx               "nginx -g 'daemon of…"   5 seconds ago       Up 2 seconds        0.0.0.0:80->80/tcp, 0.0.0.0:22->220/tcp   nginx_22_80
```

### 容器互联——bridge

#### 创建网络

```shell
➜  .ssh docker network create -d bridge my-net                      
```

#### 容器加入网络

```shell
➜  .ssh docker run -it --name busybox1 --network my-net  -d  ubuntu 
➜  .ssh docker run -it --name busybox2 --network my-net  -d  ubuntu 
```

```shell
root@02e2c31b0908:/# ping busybox1
PING busybox1 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: icmp_seq=0 ttl=64 time=0.047 ms
64 bytes from 172.18.0.2: icmp_seq=1 ttl=64 time=0.047 ms
^C--- busybox1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.047/0.047/0.047/0.000 ms
root@02e2c31b0908:/# ping busybox2
PING busybox2 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: icmp_seq=0 ttl=64 time=0.077 ms
64 bytes from 172.18.0.3: icmp_seq=1 ttl=64 time=0.100 ms
64 bytes from 172.18.0.3: icmp_seq=2 ttl=64 time=0.048 ms
64 bytes from 172.18.0.3: icmp_seq=3 ttl=64 time=0.049 ms
^C--- busybox2 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.048/0.068/0.100/0.000 ms

```

#### DNS

查看container的挂载

```shell
root@02e2c31b0908:/# mount |  grep -E 'hostname|resolv.conf|hosts'
/dev/sda4 on /etc/resolv.conf type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/sda4 on /etc/hostname type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/sda4 on /etc/hosts type ext4 (rw,relatime,errors=remount-ro,data=ordered)
```

wyny@x270 # cat /etc/docker/daemon.json 
{"registry-mirrors": ["http://0340e178.m.daocloud.io"]}container依赖于宿主机的DNS解析

```shell
wyny@x270 # cat /etc/docker/daemon.json 
{
"dns" : [
"114.114.114.114",
"8.8.8.8"
]
}
```

#### HOSTNAME

```shell
wyny@x270# docker run -d -p 80:80  --hostname=host1  --dns=114.114.114.114  --name host1  nginx 
4902ba5bcc869d20f1d79e8cc9ccac4866d19bebbe087f0f86e25a7233705427
wyny@x270# docker container ls 
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
4902ba5bcc86        nginx               "nginx -g 'daemon of…"   8 seconds ago       Up 5 seconds        0.0.0.0:80->80/tcp   host1
4b17dbab82df        ubuntu              "/bin/bash"              3 hours ago         Up 3 hours                               busybox2
02e2c31b0908        ubuntu              "/bin/bash"              3 hours ago         Up 3 hours                               busybox1
wyny@x270# docker exec -it host1 bash 
root@host1:/# ls
bin   dev  home  lib64	mnt  proc  run	 srv  tmp  var
boot  etc  lib	 media	opt  root  sbin  sys  usr
```

## 高级网络

#### 网络配置常用参数

#### 容器访问控制

##### 容器访问外部网络

container内部开启转发

```shell
sysctl -w net.ipv4.ip_forward=1
```

或启动docker时添加参数`--ip-forward=true`



## docker compose



