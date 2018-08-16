# kubernetes

## kubernetes对象

### pod

kubernetes的最小单位，共享网络和存储

pod中多个container通过localhost进行通信

通常情况下pod不会运行一个应用的多个实例

**previleged**：特权模式，容器内可以获得等同容器外进程的权限

#### pod状态

* pending：pod已在kube API中创建，但还在准备阶段（拉取镜像、启动）
* running：pod已正产运行
* successed：pod已运行结束并成功推出，不再重启
* failed：pod中有容器异常终止
* unknown：因为一些原因无法获知pod状态



#### init容器

pod启动过程中，	init容器会按顺序依次启动，并且只有在一个init容器启动并成功退出后，才会启动下一个容器；如果启动失败，则按照restartPolicy规则重启

只有在所有init容器正常退出后，pod才会变成ready

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

#### pause容器

* 在pod中担任命名空间共享的基础
* 启用pid命名空间，开启init进程

pause容器启动后，创建一个命名空间，pod中其他应用通过加入相同的命名空间**获得共享的网络和存储**

```shell
# 启动一个pause容器
docker run -d --name pause -p 8880:80 jimmysong/pause-amd64:3.0
# 启动一个nginx，加入到pause容器的命名空间中
docker run -d --name nginx -v `pwd`/nginx.conf:/etc/nginx/nginx.conf --net=container:pause --ipc=container:pause --pid=container:pause nginx
# 启动一个博客系统，加入到pause容器的命名空间中
docker run -d --name ghost --net=container:pause --ipc=container:pause --pid=container:pause ghost
```

