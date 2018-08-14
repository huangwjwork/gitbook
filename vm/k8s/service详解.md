# service详解

## service 配置文件详解

```yaml
# 版本
apiVersion: v1
# 类型service
kind: Service
# 元数据
metadata:
    # service name
    name: string
    # 所属namespace
    namespace: string
    # 标签键值对，列表
    labels:
      - name: string
    # 注解
    annotations:
      - name: string
# 详解
spec:
    # pod选择的标签列表
    selector: []
    # service类型，指定service访问类型，默认clusterIP，还有NodePort(直接采用宿主机IP+端口)，LoadBalance（指定外部负载器的IP地址，并同时定义nodePort和clusterIP）
    type:
        string
    # service的cluster IP
    clusterIP: string
    sessionsAffinity: 是否支持session
    # 端口配置
    ports:
      - name: string
        protocol: string
        port: int
        # pod端口
        targetPort: int
        # 宿主机端口
        nodePort: int
# 当spec.type=NodePort时，指定的外部负载均衡地址
status:
    # 外部负载均衡器配置
    loadBalancer:
        ingress:
            ip: string
            hostname: string
```

## 负载模式

* roundrobin 轮询
* sessionaffinity IP哈希

默认情况下采用roundrobin，但可以通过service.spec.sessionAffinity设置为clientIP启用sessionAffinity

某些情况下，也可以**不使用service负载均衡**，通过labelselector将后端的pod列表返回给调用的客户端

```yaml
clusterIP: None
selector: 
    api: nginx
```

## 集群外部服务的连接方式

当集群内部需要与外部通信时，可以将外部服务创建为一个没有selector的service，手动添加endpoint，endpoint和service同名

```yaml
kind: Service
apiVersion: v1
metadata:
    name: my-service
    spec:
    ports:
      - protocol: TCP
        port: 80
        targetPort: 80
        
---

kind: Endpoints
apiVersion: v1
metadata:
    name: my-service
subsets:
  - addresses:
      - IP: 1.2.3.4
    ports:
      - port: 80
```

## 集群外部访问pod和service

### 将container端口映射给node

#### hostPort

通过设置hostport将containerport的内容通过自定义的hostport映射到node

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: webapp
    labels:
      - app: webapp
spec:
    containers:
      - name: webapp
        image: tomcat
        ports:
          - containerPort: 8080
            hostPort: 8081
```

#### hostNetwork=true

如果设置hostport=true，则pod中容器端口号会被直接映射到node

若指定hostPort，则hostPort必须等于containerPort

若未指定hostPort，则hostPort默认等于containerport

```yaml
apiVersion: v1
kind: Pod
metadata: webapp
    name: webapp
    labels:
        app: webapp
spec:
    hostNetwork: true
    containers:
      - name: webapp
        image: tomcat
        imagePullPolocy: Never
        ports:
          - containerPort: 8080
```

### 将service端口映射给node

### 通过设置nodePort映射给端口

```yaml
apiVersion: v1
kind: Service
metadata:
    name: my-service
spec:
    selector:
        app: MyApp
    ports:
      - protocol: TCP
        port: 80
        targetPort: 9376
        nodePort: 30061
```

### loadbalance

```yaml
apiVersion: v1
kind: Service
metadata:
    name: my-service
spec:
    selector:
        app: MyApp
    ports:
      - protocol: TCP
        port: 80
        targetPort: 9376
        nodePort: 30061
    clusterIP: 10.0.171.239
    loadBalancerIP: 78.11.24.19
    type: LoadBalancer
status:
    loadBalancer：
        ingress:
          - ip: 146.148.47.155
```

## DNS原理



## ingress

普通的service只能实现loadbalance，Ingress可以将不同的URL请求转发到不同的service，实现http层的业务路由机制

类似nginx的location+proxypass

* http://mywebsite.com/api 路由到api service
* http://mywebsite.com/web 路由到web service

### 创建ingress控制器

```yaml
apiversion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress
  labels: 
    app: nginx-ingress
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
    spec:
      containers:
      - name: nginx-ingress
        image: nginx-ingress
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          hostPort: 80
```

### 创建ingress规则

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
    name: mywebsite-ingrss
spec:
    rules:
      - host: mywebsite.com
        http:
            paths:
              - path: /web
                backend:
                    serviceName: webapp
                    servicePort: 80
              - path: /api
                backend:
                    serviceName: api
                    servicePort: 80
```

