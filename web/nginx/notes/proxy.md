# proxy

## 反向代理常用命令

### proxy_pass 

```nginx
server {
    location / {
        proxy_pass http://1.1.1.1:8080;
    }
}
```

#### upstream时关于http协议的写法

proxy_pass带协议，则upstream不带协议

```nginx
upstream api {
    server 1.1.1.1:8080；
    server 2.2.2.2:8080;
}

server {
    location / {
        proxy_pass http://api;
    }
}
```

upstream带协议，则proxy_pass不带协议

```nginx
upstream api {
    server http://1.1.1.1:8080；
    server http://2.2.2.2:8080;
}

server {
    location / {
        proxy_pass api;
    }
}
```

#### proxy传递URI注意事项

想要改变URI，则在proxy中配置URI

不想改变URI，则不再proxy中配置URI

```nginx
server {
    server_name server
    location / {
        proxy_pass http://proxy; # proxy_pass不带URI，则传递原本请求的URI,即访问server/aaa代理为proxy/aaa
    }
    location /aaa {
        proxy_pass http://proxy; # proxy_pass不带URI，则传递URI，即访问server/aaa代理为proxy/aaa
    }
    location /aaa {
        proxy_pass http://proxy/bbb; # proxy_pass带URI，则替换原有URI，访问server/*全部代理为proxy/bbb
    }
    location /ccc {
        proxy_pass http://proxy/; # proxu_pass带URL，则替换原有URI，访问server/*全部代理为proxy/;
    }
}
```

### proxy_hide_header

隐藏头域信息

```nginx
proxy_hide_header field;
```

### proxy_pass_header

发送部分头域信息，默认情况下nginx服务器头部不包含date，server，x-accel等来自被**代理服务器的头域信息**

```nginx 
proxy_pass_header field ; 
```

### proxy_pass_request_body

用于配置是否将客户端的请求体（请求的参数）发送给代理服务器，默认开启

```nginx 
proxy_pass_request_body on | off ; 
```

### proxy_pass_request_headers

是否将客户端请求头发送给代理服务器，默认on

```nginx
proxy_pass_request_headers on | off ; 
```

### proxy_set_header 

更改客户端的请求头，将新的请求头发送给被代理的服务器

```nginx
proxy_set_header field value ; 
```

```nginx 
proxy_set_header Host $proxy_host ; 
proxy_set_header Connection close ;
proxy_set_header Host $http_host ;
proxy_set_header Host $host:$proxy_port ; 
```

### proxy_set_body

修改客户端的请求体，并发送给代理服务器

```nginx
proxy_set_body value ; 
```

### proxy_bind  

在配置了多个被代理服务器时，指定某一台服务器处理请求

```nginx
proxy_bind address ; 
```

### proxy_connect_timeout 

nginx与被代理服务器尝试建立连接的超时时间，默认60s

```nginx
proxy_connect_timeout time ; 
```

### proxy_read_timeout

nginx向后端服务器发送read请求后等待响应的事件，默认60s

```nginx
proxy_read_timeout time ; 
```

### proxy_send_timeout

nginx向后端服务器发送write请求后等待响应的超时时间

```nginx
proxy_send_timeout time ; 
```

### proxy_http_version 

指定nginx服务器提供代理服务的http协议版本，默认1.0，1.1版本支持upstream服务器组设置中的Keepalive指令

```nginx
proxy_http_version 1.0 | 1.1 ;
```

### proxy_method 

用于设置nginx服务器请求被代理服务器时的请求方法，一般为POST或者GET，设置了该命令后，客户端的请求方法就被忽略了

```nginx 
proxy_method method ； 
```

### proxy_ignore_client_abort  

忽略客户端中断，用于设置在客户端中断网络请求时，nginx服务器是否中断对后端服务器的请求，默认off

```nginx
proxy_ignore_client_abort on | off ; 
```

### proxy_ignore_headers

是否忽略头域信息，field为忽略的头域字段 

```nginx
proxy_ignore_headers field ...;
```

### proxy_redirect

用于修改被代理服务器返回的响应头中的location和refresh

```nginx
proxy_redirect redirect replacement;
# 将后端头域（redirect）替换为修改后的头域（replacement）
proxy_redirect default;
# 将后端头域替换为location块匹配的URL
proxy_redirect off ; 
# 关闭proxy_redirect，则直接返回后端服务器的location和refresh
```

```nginx
location /server/
{
    proxy_pass http://proxyserver/source/;
    proxy_redirect default ; 
}
# default等同于如下
location /server/{
    proxy_pass http://proxyserver/source/;
    proxy_redirect http://proxyserver/source/ /server/;
}
```

### proxy_intercept_errors

开启时，如果后端你服务器状态码≥400，则nginx返回自己定义错误页

关闭时，nginx服务器直接返回后端服务器状态

```nginx
proxy_intercept——error on | off ; 
```

### proxy_headers_hash_max_size

配置存放在报文头的哈希表的容量，上限512字符

```nginx
proxy_headers_hash_max_size 512 ;  
```

### proxy_headers_hash_bucjet_size

设置nginx服务器生情存放http报文头的哈希表容量的单位大小，默认64字符

```nginx
proxy_headers_hash_bucket_size 64;
```

### proxy_next_upstream

默认情况下，服务器组遵循upstream的轮询规则；使用该指令可以在出现某些错误时将请求交由组内其他服务器处理

```nginx
proxy_next_upstream status ...;
```

* status为服务器返回状态，可以多个
  * error，与后端服务器发生连接错误
  * timeout，与后端服务器连接超时
  * invalid_header，后端服务器返回的请求头为空或者无效
  * http_400 | http_502 | http_503 等错误状态码
  * off，无法将请求发送给被代理的服务器

### proxy_ssl_session_reuse

默认启用基于SSL的https，如果错误日志中出现SSL3_GET_FINISHED:digest check failed，可以关闭该配置

```nginx
proxy_ssl_session_reuse on | off ; 
```

## proxy buffer的常用配置

proxy_buffer_size、proxy_buffers、proxy_busy_buffers_size都是针对每一条请求

每一次请求，nginx服务器尽可能多的从后端服务器接收响应数据，若响应数据超过buffer，则写入部分临时文件，一次响应数据接收完成或者buffer装满后，才向客户端传输数据

每一个proxy buffer装满数据后，向客户端发送数据直至全部发送完成，过程中都处于busy状态，期间对他进行的其他操作都会失败

当proxy buffer关闭时，nginx只要接收到后端服务器响应数据就会同步传递给客户端，本身不再读取完成的响应数据

### proxy_buffering

是否开启proxy buffer，也可以通过在HTTP响应头鱼x-accel-buffering头域设置yes或者no

```nginx
proxy_buffering on |  off ; 
```

### proxy_buffers

用于配置接收一次后端服务器响应数据的proxy buffer个数和每个buffer的大小

```nginx
proxy_buffers number size ; 
```

size一般为内存分页大小

一次后端服务器相应数据的proxy buffer的总大小为number*size

### proxy_buffer_size 

用于配置从后端服务器获取的第一部分响应数据的大小，一般包含HTTP响应头

```nginx
proxy_buffer_size size ;
```

一般与proxy_buffers设置保持一致或者更小

### proxy_busy_buffers_size size

用于限制同事处于busy的Proxy buffer的大小

### proxy_temp_path

配置用于临时存放后端服务器大体积响应数据的本地磁盘路径

```nginx
proxy_temp_path path [level 1  [ level 2 [ level 3 ] ] ] ; 
```

level N：设置在path下第几级hash目录下存放临时文件 1 2 3 为嵌套关系

```nginx
proxy_temp_path /nginx/proxy_web/spool/proxy_temp  1 2 ; 
```

配置为level2 二级目录，则目录如下

```shell
/nginx/proxy_web/spool/proxy_temp/1/10/10101010101
```

### proxy_max_temp_file_size 

用于限制所有临时文件的总大小

```nginx
proxy_temp_file_write_size size；
```

### proxy_temp_file_write_size

用于配置同时写入临时文件的数据量的总大小

```nginx
proxy_temp_file_write_size size ;
```

## proxy_cache的常用配置

### buffer与cache的区别

* buffer，缓冲，主要用于传输速率不同步或者优先级不相同的设备之间传递数据，一般通过对一方数据进行临时存放，再同一发送给另一方
* cache，缓存，将已有的数据在内存中建立缓存数据，提高数据的访问效率，对于过期不用的缓存可以随时销毁，但不会销毁硬盘上的数据

### proxy_buffer与proxy_cache

* proxy_buffer实现后端服务器与客户端的异步传输
* proxy_cache实现nginx服务器对客户端的快速响应；将后端服务器回复到nginx的数据缓存在nginx本地，当客户端请求想用数据时，nginx服务器可以直接从本地检索数据返回给用户，减少与后端服务器交互的时间，提高响应速度

proxy_cache需要在proxy_buffer开启的情况下才能发挥作用

### proxy_cache常用命令

#### proxy_cache

用于配置一块公用的内存区域，用于存放缓存的索引数据

```nginx
proxy_cache zone | off ; 
```

* zone，设置用于存放缓存索引的内存区域的名称
* off，关闭proxy_cache功能

proxy cache会检测候选服务器响应数据的HTTP头中的cache-control，expires。当cache-control值为no-cache，no-store，private，max-age=0时，expires头域包含一个过期时间时，数据不会被nginx服务器缓存，这是为了避免私有数据被其他客户端得到

#### proxy_cache_bypass

当写在proxy_cache_bypass中的任意一个变量不为空且不为0，响应数据不从缓存中获取

```nginx
proxy_cache_bypass $cookie_nocache $arg_nocache $arg_comment $http_pragma $http_authorization ;
```

#### proxy_cache_key

用于设置nginx服务器在内存中为缓存数据建立索引时的关键字，通常配置如下字段

```nginx
proxy_cache_key "$scheme$proxy_host$uri$is_args$args"
```

#### proxy_cache_lock

