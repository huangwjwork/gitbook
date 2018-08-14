# pod详解

## pod配置文件示例

```yaml
# API版本
apiVersion: v1  
# 控制器类型
kind: pod
# 元数据
metadata:
    # pod名
    name: string
    # pod所属的命名空间
    namespace: string
    # pod的label键值对，列表
    labels:
      - name: string
    # 自定义注解键值对，列表
    annotations:
      - name: string
# pod详细定义
spec:
    # 容器列表
    containers:
        # 第1个容器名称
      - name: string
        # 容器镜像
        image: string
        # 镜像拉取的策略，always每次都重新下载镜像；IfNotPresent如果本地有就用本地，没有时下载镜像；never只使用本地镜像
        imagePullPolicy: [ Always | Never | IfNotPresent ]
        # 容器启动命令，不指定则使用镜像打包是使用的启动命令
        command: [string]
        # 启动参数
        args: [string]
        # 容器工作目录
        workingDir: string
        # 容器存储卷配置列表
        volumeMounts:
            # 卷1
          - name: string
            # 挂载的容器目录
            mountPath: string
            # 是否只读，默认为读写
            readonly: boolean
        # 容器需要暴露的端口列表
        ports:
            # 第一个端口配置
          - name: string
            # 容器监听的端口
            containerPort: int
            # 容器所在node需要监听的端口，默认与containerPort相同，设置后，同一node不可以存在多副本
            hostPort: int
            # 端口协议，TCP/UDP
            protocol: string
        # 容器的环境变量，列表
        env:
            # 第一个环境变量，变量名和值
          - name: string
            value: string
        # 容器资源配置
        resources:
            # 资源限制
            limits:
                cpu: string
                memory: string
            # 资源请求
            requests:
                cpu: string
                memory: string
        # 容器存活探针
        livenessProbe:
            # 执行命令检测是否存活
            exec:
                command: [string]
            # 发起HTTP请求检测是否存活
            httpGet:
                path: string
                port: number
                host: string
                scheme: string
                httpHeaders:
                  - name: string
                    value: string
            # 检测端口监听
            tcpSocket:
                port: number
            initialDelaySeconds: 0
            timeoutSeconds: 0
            periodSeconds: 0
            successThreshould: 0
            failureThreshould: 0
        securityContext:
            privileged: false
    # 重启策略
    restartPolicy: [ Always | Never | Onfailure ]
    # 选择启动pod的node标签
    nodeSelector: object
    # 
    imagePullsecrets:
      - name: string
    # 
    hostNetwork: false
    # pod共享存储配置，列表
    volumes:
      - name: string
        # emptyDir模式，占用内存，跟随pod生命周期
        emptyDir: {}
        # node本地路径
        hostPath:
            path: string
        # 类型为secret的存储卷，表示挂载集群预定义的secret对象到容器内部
        secret:
            secretName: string
            items:
              - key: string
                path: string
        # configMap
        configMap:
            name: string
            items:
              - key: string
                path: string                
```

## pod基本用法

* 与docker一样，前台执行（后台nohup执行后，pod认为nohup执行结束，会关闭pod，若定义了rc则无限重启）,无法前台的应用可以使用supervisor
* 紧耦合的应用建议放置在同一个pod
* 同一pod的容器可以通过localhost通信

## 静态pod

由***kubelet***运行在特定node上的pod，不受API管理，无法与RC、deployment、daemonset关联

创建静态pod有两种方式，配置文件和HTTP

* 配置文件

  将yaml配置文件防止在kubelet配置文件目录中，然后启动kubelet

  kubelet --config=/etc/kubelet.d/

## pod共享volume

```yaml
APIVersion: v1
kind: Pod
metadata:
	name: volume-pod
spec:
	containers:
	  - name: tomcat
		image: tomcat
		ports: tomcat
		  - containerPort: 8080
		volumeMounts:
		  - name: app-logs
		    mountPath: /usr/local/tomcat/logs
	  - name: busybox
	    image: busybox
		command: ["sh","-c"."tail -f /logs/catalina*.log"]
		volumeMounts:
		  - name: app-logs
		    mountPath: /logs
	volumes:
	- name: app-logs
	  emptyDir: {}
```

pod中配置一个共享存储卷app-logs

Tomcat挂载在/usr/local/tomcat/logs写日志

busybox挂载在/logs读取日志

## configMap

### 创建方法

1. yaml配置文件
1. kubectl create configamp $NAME --from-file=[key1=]source1 --from-file=[key2=]source2
1. kubectl create configmap $NAME --from-literal=key1-value1 --from-literal=key2-value2

### 使用方法

#### 生成容器内的环境变量

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
    name: cm-appvars
data:
    apploglevel: info
    appdatadir: /var/data
```



#### 设置容器启动命令的启动参数

#### 以volume形式挂载为容器内部的文件或目录

写了一个挂载configMap的pod，tail配置文件保持前台，查看pod日志即可看到configmap映射是否成功

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
    name: configmap-for-yaml
data:
    application.yaml.key: |-
        apiVersion: v1
        kind: ConfigMap
        metadata:
            name: cm-appvars
        data:
            apploglevel: info
            appdatadir: /var/data

---

apiVersion: v1
kind: Pod
metadata:
    name: configmap-pod

spec:
    containers:
      - name: configmap-pod
        image: busybox
        volumeMounts:
          - name: application-yml
            mountPath: /config
        command: ['sh','-c','tail -f /config/application.yaml']
    volumes:
      - name: application-yml
        configMap: 
            name: configmap-for-yaml
            items:
              - key: application.yaml.key
                path: application.yaml
```

查看pod的tail后的log

```shell
[k8s@kube-node1 ~]$ kubectl logs configmap-pod
apiVersion: v1
kind: ConfigMap
metadata:
    name: cm-appvars
data:
    apploglevel: info
```

## pod生命周期和重启策略

### pod状态

|  状态值   | 描述                                                         |
| :-------: | :----------------------------------------------------------- |
|  pending  | APIserver已创建pod，但pod内还有容器的镜像没有创建            |
|  running  | pod内所有容器都已创建，切至少一个容器处于运行状态、正在启动状态和正在重启状态 |
| succeeded | pod内所有容器均执行成功并退出，且不再重启                    |
|  failed   | pod内所有容器已退出，但至少还有一个容器退出为失败状态        |
|  unknown  | 无法获取pod状态，可能由于网络通信不畅导致                    |

### pod重启策略

* Always	容器失效时自动重启
* OnFailure 容器中支运行且退出码部位0时，有kubelet自动重启该容器
* Never 不论容器运行状态如何，kubelet都不会重启该容器

重启的时间间隔以sync-frequency乘2计算，1、2、4、8等，最长延时5分钟，并在成功重启后10分钟后重置该时间

不同的控制器对pod的充气策略要求如下：

* RC和DaemonSet：Always
* Job： OnFailure或Never
* kubelet：pod失效时自动重启

## pod健康检查

### LivenessProbe

用于判定容器是否存活，如果探测到容器不健康，则kubelet杀掉改容器，并根据容器的重启策略做相应的处理；如果不配置LivenessProbe，则认为容器返回值永远是success

#### ExecAction

通过执行命令的返回值判定容器健康状态

```yaml
apiVersion: v1
kind: Pod
metadata:
    labels: 
        test: liveness
    name: liveness-exec
spec:
    containers:
      - name: liveness
        image: busybox
        args:
          - /bin/sh
          - -c
          - echo ok > /tmp/health;sleep 10;rm -rf /tmp/health;sleep 60
        livenessProbe:
            exec:
                command:
                  - cat
                  - /tmp/health
            initialDelaySeconds: 30 
            timeoutSeconds: 1
```

describe这个pod会发现在不断的重启

#### TCPsocketProbe

通过容器的IP和端口检查，能简历TCP链接，则表明容器健康

```yaml
livenessProbe:
    tcpSocket:
        port: 80
```

#### HTTPGetAction

通过get方法，如果相应的状态码大于等于200且小于等于400，则认为容器状态健康

```yaml
livenessProbe：
    httpGet:
    path: /_status/healthz
    port: 80
    initialDelaySeconds: 30
    timeoutSeconds: 1
```



### ReadinessProbe

## node定向调度

* nodeSelector 通过label键值对选择node

  ```yaml
  selector:
      name: frontend
  ```

* nodeAffinity

  selector升级版，添加了in，noting，exists，doesnotexist，gt，lt等操作符

  * RequiredDuringSchedulingRequiredExection：类似于nodeselector，node不满足条件时，系统将从该node上移除之前调度的pod
  * RequiredDuringSchedulingIgnoredDuringExection，类似RequiredDuringSchedulingRequiredExection，区别在于node不满足条件时，系统不一定从该node上移除之前调动的pod
  * PreferredDuringSchdulingIgnoredDuringExection：满足条件的node优先调度，不满足的node不一定移除


在当前版本中，需要在pod的metadata中设置nodeaffinity

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: with-lables
    nanotations:
        scheduler.alpha.kubernetes.io/affinity
        .......
        
        
```

## DaemonSet

每个node上只运行一个的特殊pod

```yaml
apiVersions/v1beta1
kind: DaemonSet
metadata:
    name: fluentd-cloud-logging
    namespace: kube-system
    labels:
        k8s-app: fluentd-cloud-logging
    spec:
        template:
            metadata:
                namespace: kube-system
            labels:
                k8s-app: fluentd-cloud-logging
            spec:
                container:
                  - name: fluentd-cloud-logging
                    image: fluentd-elasticsearch:1.17
                    resource:
                        limits: 
                            cpu: 100m
                            memory: 200Mi
                    env:
                      - name: FLURNTD_ARGS
                        value: -q
                    volumeMounts:
                      - name: varlog
                        mountPath: /var/logging
                        readOnly: false
                      - name: varlog
                        mountPath: /var/log
                        readOnly: false
                Volumes:
                  - name: containers
                    hostpath:
                        path: /var/lib/docker/containers
                  - name: varlog
                    hostpath:
                        path: /var/log
```

## job

* job Template Expansion: 一个job对应一个待处理的workitem，有几个workitem就产生几个独立的job，通常适合workitem数量较少、每个workitem需要处理大量的数据
* queue with pod per work item，采用一个任务队列存放workitem，一个job对象作为消费者去完成，在这种模式下，job会启动N个pod，每个pod对应一个workitem
* queue with variable pod count：采用一个任务队列存放workitem，一个job对象作为消费者去完成这些workitem，但job启动的pod数是可变的

## 滚动升级

```shell
kubectl rolling-update redis-master -f redis-master-controller-v2.yaml
```

