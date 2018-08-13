# gzip压缩

gzip压缩页面需要浏览器和服务器双方支持，即服务器端压缩，浏览器端解压并解析。压缩页面后，传输流量小，传输速度更快

我的配置如下，压缩效果很明显：

```shell
gzip on;
gzip_comp_level 9;
gzip_min_length 1024;
gzip_types       text/plain application/javascript text/css application/xml application/json;
```

查看header信息中`Content-Encoding: gzip`

```shell
$ curl -I -H "Accept-Encoding: gzip, deflate"  https://huangwj.app/search_index.json
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0HTTP/1.1 200 OK
Server: nginx/1.12.2
Date: Tue, 12 Jun 2018 06:28:11 GMT
Content-Type: application/json
Last-Modified: Mon, 11 Jun 2018 09:24:44 GMT
Connection: keep-alive
ETag: W/"5b1e3fdc-a2cf6"
Content-Encoding: gzip
```

压缩效果很nice，压缩前search_index.json660K左右，压缩后80K

## ngx_http_gzip_module模块处理的9个指令

### gzip

用于启用或关闭gzip功能，默认off

```shell
gzip on | off;
```

### gzip_buffers

设置gzip压缩文件使用的缓存空间大小

* number，指定nginx服务器需要向系统申请缓存空间的个数，默认number*size=128
* size，指定每个缓存空间的大小，一般取内存页一页的大小，4KB或8KB

```shell
gzip_buffers number size;	# 32 4K | 16 8K 
```

### gzip_comp_level

指定压缩级别，1-9从低到高，压缩程度从低到高，压缩效率从高到底（时间、计算效率）

```shell
gzip_comp_level level;
```

### gzip_disable

针对不同种类客户端发起的请求，选择性开启或关闭gzip功能

```shell
gzip_disable regex ...;
```

### gzip_http_version

针对不同HTTP协议版本，选择性开启gzip功能，用于设置开启gzip功能的最低http协议版本

```shell
gzip_http_version 1.1;
```

默认1.1版本

### gzip_min_length

启用压缩的下限，过小的页面压缩没有意义，单位字节

```shell
gzip_min_length 1024;
```

默认20，设置为0是表示不管响应页面大小如何通通进行压缩，建议1KB以上

### gzip_proxied

反向代理时后效，用于设置是否对后端服务器返回的结果进行gzip压缩

```shell
gzip_proxied off | expired | no-cache | no-store | private | no_last_modified | no_etag | auth | any ...;
```

* off ，关闭nginx对后端服务器返回结果的gzip压缩，默认关闭
* expired，后端服务器响应头部包含用于只是响应数据过期时间的expired头域时，启用gzip
* no-cache，后端服务器响应页头部包含用于通知所有缓存机制是否缓存cache-control头域，且指令值为no-cache时，启用gzip
* no-store，后端服务器响应页头部包含用于通知所有缓存机制是否缓存cache-control头域，且指令值为no-store时，启用gzip
* private，后端服务器响应页头部包含用于通知所有缓存机制是否缓存cache-control头域，且指令值为private时，启用gzip
* no_last_modified，后端服务器响应页头部不包含用于指明需要获取数据最后秀爱时间的last-modified头域时，启用gzip
* no-etag，当后端服务器响应页头部不包含用于标示被请求变量的实体值etag头域时，启用对响应数据的gzip压缩
* auth，后端服务器响应页头部包含用于标示HTTP授权证书的authorizationl头域，且指令值为private时，启用gzip
* any，无条件对后端服务器响应数据的gzip

### gzip_types

根据MIME类型选择性开启gzip

```shell
gzip_types mime-type ...;
```

gzip开启时，默认对text/html进行gzip压缩，可以按照需求添加压缩类型，也可以压缩所有“*”

```shell
gzip_types text/plain application/javascript text/css text/html application/xml;

gzip_types * ; 
```

### gzip_vary

用于设置是否在响应头部添加"Vary: Accept-Encoding"，告诉接收方接收到的数据是否为压缩数据。对于不慎不支持gzip压缩的客户端浏览器很有用

```shell
gzip_vary on | off ;
```

也可以用add_header得到相同效果

```shell
add_header Vary Accept-Encoding gzip ; 
```

**注意：该指令存在bug，会导致IE4级以上浏览器的数据缓存功能失效**

## ngx_http_gzip_static_module模块处理的指令

可选模块，使用前需要--with-http_gunzip_module

ngx_http_zip_static_module模块主要负责搜索和发送经过gzip压缩过的数据，这些数据以”GZ“作为后缀名存储在服务器。如果请求的数据被压缩过，且客户端支持gzip，直接返回压缩后数据

* ngx_http_gzip_static_module，使用静态压缩，在HTTP响应头部包含Content-Length指明报文长度，服务器可确定相应数据长度
* ngx_http_gzip_module，使用chunked编码的动态压缩，主要适用于服务器无法确定响应数据长度的情况，比如大文件下载，需要实时生成数据长度

### 相关指令

* `gzip_static on | off | always;`
  * on，开启模块功能
  * off，关闭模块功能
  * always，一直发送gzip预压缩文件，不检查客户端是否支持gzip
* `gzip_proxied expired | no-cache | no-store | private auth ;`

**注意：该模块下的gzip_vary，只给未压缩的内容添加'Vary:Accept-Encoding'，如果要给所有响应头添加头域，可以通过nginx配置add_header指令实现**

## ngx_http_gunzip_module模块处理的2个指令

可选模块，使用前需要--with-http_gunzip_module

默认关闭，开启式如果客户端不支持gzip，则返回解压后的数据，如果支持gzip，则仍然返回压缩数据

**gunzip_static实际测试中没有该指令，官方文档也查不到！！！！**

http://nginx.org/en/docs/http/ngx_http_gunzip_module.html

### gunzip

### gunzip_static

```shell
gunzip_static on | off ;
```

### gunzip_buffer

类似ngx_http_gzip_module中的gzip_buffers

```shell
gunzip_buffers number size ; 
```

默认number*size=128

## gzip配置实例

```shell
gzip  on ;	#开启gzip功能
gzip_min_length ;	#开启gzip的文件大小
gzip_buffers 4 16K；	#申请4个缓存空间，每个16K
gzip_comp_level 2；	#压缩级别2
gzip_types text/plain applicatio/json text/css application/xml application/javascript;
gzip_vary on;
gunzip_static on ; 
```

建议配置在HTTP块，全局开启压缩，不压缩的server可以单独gzip off

```shell
server {
    listen 8082 ; 
    server_name 192.168.1.1 ;
    gzip off ;
}
```

## nginx与其他web服务器协作gzip

nginx作为反向代理时建议关闭后端服务器gzip，开启nginx gzip，否则可能出现页面异常



