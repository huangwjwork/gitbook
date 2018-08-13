# Rewrite

[TOC]

## Upstream命令-负载

### upstream

```shell
upstream name {...} ; 
```

name为服务器组的组名（别名），花括号中可以列出后端服务器组中包含的服务器

默认轮询（Round-Robin），轮流处理请求，如果某个服务器处理时出现错误，则切换下一个服务器，直到所有服务器报错，返回报错内容

### server

```shell
server address [patameters];
```

* address，服务器地址，可以是IP地址，主机名，域名，或者unix Domain Socket 
* parameters
  * weight=number，服务器权重，权重值高的优先处理请求
  * max_fails=number，设置请求失败次数，一定时间范围，服务器请求失败的次数搞过限定值，认为服务器down，默认值为1，设置为0则失效
  * fail_timeout，设置尝试请求某服务器的时间，即max_fails的事件范围。也用于检查服务器时候有效，如果一台服务器down，则该时间段内不会再次检查服务器状态；默认10Ss
  * backup,标记某个服务器为备用服务器，只有其他服务器down或者busy时才处理客户端请求
  * down，永久标记服务器为无效，通常与ip_hash配合使用

配置示例：

```shell
upstream backend
{
    server backend1.example.com weight=5;
    server 127.0.0.1:8080 max_fails=3 fail_timeout=30s;
    server unix:/tmp/backend3;    
}
```

### ip_hash

基于IP地址将客户端的请求定向到同一台服务器，保证客户端与服务器间会话稳定

局限：

* 不能用server中的weight变量一起使用
* nginx服务器必须是最前端的服务器，才能获取到客户端真实IP
* 客户端IP地址必须是C类地址

```shell
upstream backend {
    ip_hash;
    server myback1.proxy.com;
    server myback2.proxy.com;
}
```

### keepalive

```shell
keepalive connections;
```

connections为nginx服务器每个worker process允许该服务器组保持的空闲网络连接数上线，超过后哦，将采用最近最少使用的策略关闭网络连接

### least_conn

配置nginx服务器使用负载均衡策，选择最少连接负载的服务器分配连接，如果有多台符合的服务器，则采用加权轮询选择这几台服务器

## Rewrite指令

rewrite依赖PCRE（peal compatible regular expressions）

### 地址重写 & 地址转发

* 地址重写：实现地址标准化，如google.com，www,google.com，google.cn，都跳向google首页
* 地址转发：将一个域名知道另一个以后站点的过程

区别：

* 地址转发后客户端地址显示不变，地址红鞋后客户端浏览器地址栏中的地址改变为服务器选择确定的地址
* 地址转发过程只产生一次网络请求；地址重写一般会产生两次请求
* 地址转发一般发生在同一站点项目内，地址重写没有该限制
* 地址转发的页面可以不用全路径表示，但地址重写到的页面必须使用完整的路径名表示
* 地址转发过程中，可以将客户端请求的request传递给新的页面，地址重写不可以
* 地址转发的速度较地址重定向快

### if

根据条件判断结果选择不同的nginx配置，可以在server块或者location块中配置该指令

```shell
if （condition） {...}
```

* 变量名条件，不为空为true，空或0false

  ```shell
  if （$slow） {
      ... #nginx配置
  }
  ```

* 变量判断

  ```shell
  if ( $request_method = POST ){
      return 405;
  }
  ```

* 正则判断，匹配则true

  ```shell
  if ( $http_user_agent ~ MSIE ) {
      ...
  }
  IF ( $HTTP_COOKIE ~* "ID=([^;]+)(?:;|$)"){
      匹配正则表达式，成功则true
      
  }
  ```

* 判断文件是否存在 -f 是否不存在  !-f

  ```shell
  if ( -f $request_filename ) {
      ...
  }
  if ( !-f $request_filename ) {
      ...
  }
  ```

* 判断目录是否存在 -d 是否不存在 !-d

* 判断目录或者文件是否存在 -e 

* 判断请求的事件是否可执行 -x

**注意：字符串不需要加引号**

### break 

跳出当前作用域

```shell
location / {
    if ($slow){
        set #id $1
        break;
        limit_rate 10k;	# break跳出当前作用域，不执行
    }
    if (condition A ){	# break上级其他作用域，顺序执行
        ...
    }    
        }
```

### return 

用于完成对请求的处理，直接返回响应状态代码，return后的配置均无效

```shell
return [text]
return code URL; # code为301,302,303,307
return URL;	# code为302,307
```

### rewrite

```shell
rewrite regex replacement [flag];
```

* rewrite接收到的URI不包含host，也不包含请求指令`?arg1=value1`
* replacement为替换URI中被截取内容的字符串
* flag
  * last，一条URI匹配规则后，被重新为一条新的URI，重新再所有location中进行匹配，提供了URI转入其他location的机会
  * break，一条URI匹配后，重写为新的URI，在本location中继续进行处理，不会转入其他的location
  * redirect，将重写后的URI返回给客户端，状态码302，临时重定向
  * permanent，重写后的URI返回给客户端，状态码301，永久重定向

### rewrite_log

```shell
rewrite_log on | off 
```

默认off，配置为on后，URL重写的日志会以notice级别输出到error_log里

### set

设置一个新的变量

```shell
set variable value
```

* variable为变量名，要用符号$标记，且不能与nginx服务器预设的全局变量同名
* value，变量的值

### uninitialized_variable_warn

用于配置使用未初始化的变量时，是偶记录警告日志：

```shell
uninitialized_variable_warn on | off
```

默认on

## rewrite的使用

### 域名跳转

```shell
sever {
    listen 80;
    server_name jump.myweb.name;
    rewrite ^/ http://www/myweb.info/;	# 域名跳转，针对jump.myweb.name的所有请求跳转到www.myweb.info
}
```

```shell
server{
    listen 80；
    server_name jump.mywebname jump.myweb.info；
    if ( $host ~ myweb\.info )	# 如果主机名正则匹配myweb.info
    {
        rewrite ^(.*) http://jump.myweb.name$1 permanent；	# 重写URI到http://jump.myweb.name/URL
    }
}
```

### 域名镜像？？

没懂和普通rewrite的区别，根据资源做分流？

### 独立域名

将一个站点根据资源分割为多个域名

```shell
server {
    listen 80;
	server_name bbs.myweb.name;
	rewrite ^(.*) http://www.myweb.name/bbs$1 last;		# 将bbs模块重写为域名bbs.myweb.name
}
server {
    listen 80;
    server_name home.myweb.name;
    rewrite ^(.*) http://www.myweb.name/home$1 last;	# 将home模块重写为域名home.myweb.name
}
```

### 目录自动添加'/'

配置index参数后

http://www.myweb.name/bbs/ 带/的可以直接查找index文件

http://ww.myweb.name/bbs 不带/不能被识别为目录，不能自动寻找index

```shell
location ^~ /bbs	# 依据匹配程度标准匹配
{
    if ( -d $request_filename ) {	# 检测请求的目录是否存在
        rewrite ^/(.*)([^/])$ http://$host/$1$2/ permanent;	# 匹配结尾没有/的目录，在结尾加上/
    }
}
```

### 目录合并

目录结构复杂不利于搜索，可以通过rewrite简化路径，比如root/server/12/34/56/78/9.html

```shell
server {
    listen 80;
    server_name www.myweb.name;
    location ^~ /server {
        rewrite ^/server-(\d+)-(\d+)-(\d+)-(\d+)\.html$ /server/$1/$2/$3/$4/html last ;
    }
}
```

### 防盗链

请求网页时，A网页中存在B站点的资源时，就存在A对B的盗链行为，B站点为了防止这种行为，就需要检测http协议中header中referer头域检测访问目标资源的源地址，当检测到referer头域中的值不是自己的站点的URL，就阻止访问

nginx中**valid_referers**用来获取referer头域中的值，并将结果赋值给全局变量`$invalid_referer`,如果referer头域中没有符合valid_referers指令配置的值，`$invalid_referer`会被赋值为1

```shell
valid_referers none | blocked | server_names | string ...;
```

* none，referer头域不存在
* blocked，检测referer头域的值被伪装或者被删除的情况
* server_names，设置一个或多个URL，检测referer头域的值是否是这些URL中的某个

```shell
server {
    listen 80;
    server_name www.myweb.name;
   	 location ~* ^.+(gif|jpg|png|swf|flv|rar|zip)$ {
    valid_referers none blocked server_names *.myweb.name;
    if ($invalid_referer){
        rewrite ^/ htttp://www.myweb.com/images/forbidden.png;
    }
    )
}
```



