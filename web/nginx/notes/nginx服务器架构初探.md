# nginx服务器架构初探

## 模块化结构

* 核心模块：进程管理、权限控制、错误日志记录
* 标准HTTP模块：支持NGINX服务器标准HTTP功能
* 可选HTTP模块：扩展标准的HTTP功能，使其能够处理一些特殊的HTTP请求
* 邮件服务模块：支持NGINX的邮件服务
* 第三方模块：扩展NGINX服务器应用

核心模块和标准HTTP模块在NGINX快速编译后就包含在NGINX中

查看NGINX源码包

```shell
[root@test nginx-1.14.0]# ll objs/
total 3724
-rw-r--r-- 1 root root   19575 May 22 11:41 autoconf.err
-rw-r--r-- 1 root root   39268 May 22 11:41 Makefile
-rwxr-xr-x 1 root root 3679067 May 22 11:41 nginx
-rw-r--r-- 1 root root    5321 May 22 11:41 nginx.8
-rw-r--r-- 1 root root    6598 May 22 11:41 ngx_auto_config.h
-rw-r--r-- 1 root root     657 May 22 11:41 ngx_auto_headers.h
-rw-r--r-- 1 root root    5725 May 22 11:41 ngx_modules.c
-rw-r--r-- 1 root root   35840 May 22 11:41 ngx_modules.o
drwxr-xr-x 9 root root    4096 May 22 11:41 src
```

查看NGINX变异后包含的的所有固有模块，这些模块声明以extern关键字修饰

```shell
[root@test nginx-1.14.0]# cat objs/ngx_modules.c |  grep extern 
extern ngx_module_t  ngx_core_module;
extern ngx_module_t  ngx_errlog_module;
extern ngx_module_t  ngx_conf_module;
extern ngx_module_t  ngx_regex_module;
extern ngx_module_t  ngx_events_module;
extern ngx_module_t  ngx_event_core_module;
extern ngx_module_t  ngx_epoll_module;
extern ngx_module_t  ngx_http_module;
extern ngx_module_t  ngx_http_core_module;
extern ngx_module_t  ngx_http_log_module;
extern ngx_module_t  ngx_http_upstream_module;
extern ngx_module_t  ngx_http_static_module;
extern ngx_module_t  ngx_http_autoindex_module;
extern ngx_module_t  ngx_http_index_module;
extern ngx_module_t  ngx_http_mirror_module;
extern ngx_module_t  ngx_http_try_files_module;
extern ngx_module_t  ngx_http_auth_basic_module;
extern ngx_module_t  ngx_http_access_module;
extern ngx_module_t  ngx_http_limit_conn_module;
extern ngx_module_t  ngx_http_limit_req_module;
extern ngx_module_t  ngx_http_geo_module;
extern ngx_module_t  ngx_http_map_module;
extern ngx_module_t  ngx_http_split_clients_module;
extern ngx_module_t  ngx_http_referer_module;
extern ngx_module_t  ngx_http_rewrite_module;
extern ngx_module_t  ngx_http_proxy_module;
extern ngx_module_t  ngx_http_fastcgi_module;
extern ngx_module_t  ngx_http_uwsgi_module;
extern ngx_module_t  ngx_http_scgi_module;
extern ngx_module_t  ngx_http_memcached_module;
extern ngx_module_t  ngx_http_empty_gif_module;
extern ngx_module_t  ngx_http_browser_module;
extern ngx_module_t  ngx_http_upstream_hash_module;
extern ngx_module_t  ngx_http_upstream_ip_hash_module;
extern ngx_module_t  ngx_http_upstream_least_conn_module;
extern ngx_module_t  ngx_http_upstream_keepalive_module;
extern ngx_module_t  ngx_http_upstream_zone_module;
extern ngx_module_t  ngx_http_write_filter_module;
extern ngx_module_t  ngx_http_header_filter_module;
extern ngx_module_t  ngx_http_chunked_filter_module;
extern ngx_module_t  ngx_http_range_header_filter_module;
extern ngx_module_t  ngx_http_gzip_filter_module;
extern ngx_module_t  ngx_http_postpone_filter_module;
extern ngx_module_t  ngx_http_ssi_filter_module;
extern ngx_module_t  ngx_http_charset_filter_module;
extern ngx_module_t  ngx_http_userid_filter_module;
extern ngx_module_t  ngx_http_headers_filter_module;
extern ngx_module_t  ngx_http_copy_filter_module;
extern ngx_module_t  ngx_http_range_body_filter_module;
extern ngx_module_t  ngx_http_not_modified_filter_module;
```

NGINX模块命名习惯：`ngx_`前缀，`_module`后缀，中间使用一个或者多个英文单词描述模块的功能。比如ngx_core_module，中间的core表明是NGINX的核心模块

### 核心模块

```shell
[root@test nginx-1.14.0]# cat objs/ngx_modules.c |  grep extern 
extern ngx_module_t  ngx_core_module;
extern ngx_module_t  ngx_errlog_module;
extern ngx_module_t  ngx_conf_module;
extern ngx_module_t  ngx_regex_module;
extern ngx_module_t  ngx_events_module;
extern ngx_module_t  ngx_event_core_module;
extern ngx_module_t  ngx_epoll_module;
```

核心模块分两类

* 主体功能：包括进程管理、权限控制、错误日志、配置解析
* 相应请求事件：事件驱动机制、正则表达式解析

### 标准HTTP模块

|模块|功能|
|:-:|:-:|
|ngx_http_core|配置端口、URI分析、服务器相应错误处理、别名控制以及其他HTTP核心事务|
|ngx_http_access_module|基于IP地址的访问控制|
|ngx_http_auth_basic_module|基于HTTP的身份认证|
|ngx_http_autoindex_module| 处理以“/”结尾的请求并自动生成目录列表 |
|ngx_http_browser_module| 解析HTTP请求头中的user-agent域的值 |
|ngx_http_charset_module| 指定网页编码 |
|ngx_http_empty_gif_module| 从内存创建一个1*1的透明GIF，可以快速调用 |
|ngx_http_fastcgi_module| 对fastCGI的支持 |
|ngx_http_geo_module| 将客户端请求中的参数转化为键值对变量 |
|ngx_http_gzip_module| 压缩请求响应，可以减少数据传输 |
|ngx_http_index_filter_module| 设置HTTP响应头 |
|ngx_http_index_module| 处理以"/"结尾的请求，如果没有找到该目录下的index页，就将请求转给ngx_http_autoindex_module模块处理；如果nginx服务器开启了ngx_http_random_index_module模块，则随机选择index页 |
|ngx_http_limit_req_module| 限制来自客户端的请求的相应和处理速率 |
|ngx_http_limit_conn_module| 限制来自客户端的链接的相应和处理速度 |
|ngx_http_map_module| 创建任意键值对变量 |
|ngx_http_log_module| 自定义access日志 |
|ngx_http_memcached_module| 对memcached的支持 |
|ngx_http_proxy_module| 支持代理服务 |
|ngx_htttp_referer_module| 过滤HTTP头中referer域值为空的HTTP请求 |
|ngx_http_rewrite_module| 通过正则表达式重新定向请求 |
|ngx_http_scgi_module| 对scgi的支持 |
|ngx_http_ssl_module| 对HTTPS的支持 |
|ngx_http_upstream_module| 定义一组服务器，可以接受来自代理、fastcgi、memcached的重定向、主要用于负载均衡 |
### 可选HTTP模块

可选HTTP模块在目前的nginx发行版本中只提供远吗，默认不编译。想要使用需编译时添加`--with-XXX`

|             模块             |                             功能                             |
| :--------------------------: | :----------------------------------------------------------: |
|   ngx_http_addition_module   |           在响应请求的页面开始或者结尾添加文本信息           |
| ngx_http_degradation_module  |      在低内存的情况下允许nginx服务器范围444或者204错误       |
|     ngx_http_perl_module     |              在nginx配置文件中可以使用Perl脚本               |
|     ngx_http_flv_module      | 支持将flash多媒体信息按照刘文建传输，可以根据客户端指定的开始位置返回flash |
|    ngx_http_geoip_module     |             支持解析基于GeoIP数据库的客户端请求              |
|  ngx_http_perftools_module   |                 支持google performance tools                 |
|     ngx_http_gzip_module     |            支持在线实施压缩响应客户端的输出数据流            |
| ngx_http_gzip_static_module  | 搜索并使用与压缩的以`.gz`为后缀名的文件代替一般文件响应客户端请求 |
| ngx_http_image_filter_module |          支持改变JPEG、GIF、PNG图片的尺寸和旋转方向          |
|     ngx_http_mp4_module      |        支持H.264/AAC编码的多媒体信息（MP4、M4V、M4A）        |
|   ngx_random_index_module    | nginx接收到以`"/"`结尾的请求时，对响应目录下随机选择一个文件作为index文件 |
| ngx_http_secure_link_module  |                  支持对请求链接的有效性检查                  |
|     ngx_http_ssl_module      |                        支持HTTPS/SSL                         |
| ngx_http_stub_status_module  | 支持返回nginx服务器的统计信息，包括处理链接的数量，链接成功的数量，处理的请求书，读取和返回的header信息数等信息 |
|     ngx_http_sub_module      |             使用指定的字符串替换响应信息中的信息             |
|     ngx_http_dav_module      | 支持HTTP协议和webDAV协议中的PUT、DELETE、NKCOL、COPY和MOVE方法 |
|     ngx_http_xslt_module     |     将XML响应信息使用XSLT（扩展样式表转换语言）进行转换      |

### 邮件服务模块

默认编译时不编译邮件服务模块

* ngx_mail_core_module
* ngx_mail_pop3_module
* ngx_mail_imap_module
* ngx_mail_smtp_module
* ngx_mail_auth_http_module
* ngx_mail_proxy_module
* ngx_mail_ssl_module

### 第三方模块

wiki站点

## nginx服务器的web请求处理机制

web服务器和客户端是一对多的关系，要实现并行处理请求，有三种方式可选：多进程、多线程、异步

### 多进程方式

是指服务器没收到一个客户端请求，就由服务器主进程生成一个子进程与客户端建立连接进行交互，知道连接端口子进程结束

* 优点

  实现简单，各子进程相互独立，处理过程不受干扰，稳定性好；处理结束后占用资源会被操作系统回收

* 缺点

  操作系统建立子进程需要进行内存复制等操作，资源和时间上产生一定的额外开销，当处理大量并发请求时，系统资源压力大

**初期的Apache采用这种多进程方式，为应对大量并发请求，它采用预生成进程，即在客户端请求未到来前生成好子进程，处理结束后也不释放进程，等待下一个请求**

### 多线程方式

多线程与多进程类似，当服务器收到一个客户端请求时，由服务器主进程派生出一个县城出来和改客户端进行交互。

* 优点

  由于操作系统产生一个县城的开销远远小于产生一个进程的开销，所以多线程减轻了web服务器对系统资源的要求。

* 缺点

  多个线程处于同一进程，可以访问同样的内存空间，彼此相互影响；开发者需要对内存进行管理，增加出错风向；服务器需要长时间连续不停运转，错误的逐渐累积会对服务器产生重大影响

**IIS服务器使用多线程方式，稳定性良好，但经验丰富的web服务器管理员还是会定期检查和重启服务器**

### 异步方式

先从网络通信层面分析同步、异步、阻塞、非阻塞：

* 同步机制

  是指发送请求后，收到接收方返回的相应后，继续发送下一个请求

* 异步机制

  发送方发出一个请求，不等待接收方相应请求，就继续发送下一个请求

同步机制中，所有的请求在服务器端得到同步，发送方和接收方对请求的处理步调是一直的；异步机制中，所有来自发送方得请求形成一个队列，接收方处理完成后通知发送方

* 阻塞

  socket本质是IO操作。socket阻塞调用方式为：调用结果返回前，当前线程从运行状态变成挂起，一直等到调用结果返回之后，才进入就绪状态，获取CPU后继续执行

* 非阻塞

  如果调用结果不能发上返回，当前线程也不会被挂起，而是立即返回执行下一个调用

---

* 同步阻塞方式

  发送方想接收方发送请求后，一直等待相应，接收方处理请求时进行的IO操作如果不能马上得到结果，就一直等到返回结果后，才响应发送方，期间不能进行其他工作。

* 同步非阻塞方式

  发送方想接收方发送请求后，一直等待相应；接收方处理请求时进行的IO操作如果不能马上得到结果，就立即返回，去做其他事情，但与有没有得到请求处理结果，不相应发送方，发送方一直等待；一直到IO操作完成后，接收方活得结果相应发送方后，接收方才进入下一次请求过程，在实际中不适用这种方式。

* 异步阻塞方式

  发送方方向接收方发送请求后，不用等待相应，可以接着进行其他工作；接收方处理请求时进行的IO操作如果不能马上得到结果，就一直等到返回结果后，才相应发送方，期间不能进行其他工作，实际中不使用

* 异步非阻塞方式

  发送方向接收方发送请求后，不用等待响应，可以继续其他工作；接收方处理请求时进行的IO操作如果不能马上得到结果，也不等待，而是马上返回去做其他事情。当IO操作完成以后，将完成状态和结果通知接收方，接收方再相应发送方。

  **异步非阻塞是通信效率最高的一种**

### nginx如何处理请求

nginx结合多进程机制和异步非阻塞机制对外提供服务。启动nginx后，会产生一个主进程（master process nginx）和多个工作进程（worker process），可以在配置文件中指定worker process数量。只有工作进程才用于处理客户端请求。

```shell
[root@test ~]# ps -ef |grep  -i nginx 
root      1425     1  0 May22 ?        00:00:00 nginx: master process nginx
elk       1426  1425  0 May22 ?        00:00:00 nginx: worker process
```

**nginx服务器进程模型有两种**

* single模型

  单进程，性能差，一般不使用

* master-worker模型

  即master-slave模型，常用，工作进程即slave

每个worker process都是用异步非阻塞方式，客户端请求数量增长，万国府在繁重时，nginx服务器使用多进程机制能够保证不增长系统资源的压力；同事异步非阻塞方式减少了工作进程在IO调用上的阻塞延迟，保证了不降低对请求的处理能力

### nginx的事件处理机制

nginx采用异步非阻塞方式处理请求，当IO调用完成后，需要通知工作进程。有两种方案：

1. worker process每隔一段事件去检查IO的运行状态，这样会增大资源开销
1. **IO处理完毕后主动通知worker process，nginx使用这种事件处理机制**

**select / poll / epoll / kqueue**等事件驱动模型都用来支持第二种解决方案，可以让进程同时处理多个并发请求，不用关心IO调用的具体状态，事件准备好后就通知工作进程事件已经就绪。

## nginx服务器的事件驱动模型

### 事件驱动模型概述

事件驱动就是在持续事物管理过程中，有当前时间点上出现的事件引发的调用可用应用资源执行相关人物，解决不断出现的问题，防止事物堆积的一种策略。

事件驱动模型组成：

* 事件收集器

  收集所有事件，来自用户的（鼠标键盘输入等），来自硬件的（时钟事件）、来自软件（操作系统、应用程序）

* 事件发送器

  事件发送器负责将收集器收集到的事件分发到目标对象中，即发送给事件处理器

* 事件处理器

  主要负责具体时间的响应

事件发送器每传递过来一个请求，目标对象就将其放入一个待处理时间列表，使用非阻塞IO的方式调用事件处理器来处理请求

事件驱动处理库又被称作多路IO服用方式发，常见的有select模型、poll模型、epoll模型，nginx还支持rtsig模型、kqueue模型、dev/poll模型和eventport模型。

### 事件驱动模型

后期补充

#### select库

#### poll库

#### epoll库

#### rtsig库

#### 其他事件驱动模型

## 设计架构概览

### 服务器架构

### 服务进程

* master process（主进程）

  外界通信、内部管理

  * 读取nginx配置文件并验证其有效性和正确性
  * 建立、绑定和关闭socket
  * 按照配置生成、管理和结束工作进程
  * 接收外界指令，比如中期、升级即退出服务器等指令
  * 不中断服务、实现平滑升级，升级失败进行回滚处理
  * 开启日志文件、获取文件描述符
  * 编译和处理Perl脚本

  

* worker process（工作进程）

  处理用户请求

  * 接收客户端请求
  * 将请求一次送入各个功能模块进行过滤处理
  * IO调用，获取响应数据
  * 与后端服务器通信，接受后端服务器处理结果
  * 数据缓存，访问缓存索引、查询和调用缓存数据
  * 发送请求结果、响应客户端请求
  * 接收主程序指令、比如重启、升级和退出等指令

* cache loader & cache manager（缓存索引重建及管理进程）

  * cache loader（缓存索引重建）

    每1分钟1次，由master process生成，在缓存元数据重建完成后就自动退出

    根据本地此案的缓存文件在内存中建立索引元数据库

  * cache loader（缓存索引管理）

    主进程整个生命周期，对缓存索引进行管理

    判定索引元数据是否过期

### 进程交互

  1. master-worker交互

     主进程指向工作进程的单向管道

  1. worker-worker交互

     通过master进程交互

### run loops 事件处理循环模型

非自动，需要在设计代码过程中，在适当的时候启动run-loop机制对输入的事件作出响应。

该模型师一个集合，每个元素就是一个run-lopp。一个run-loop可运行在不同模式下， 其中可以包含他所坚挺的输入事件源、定时器以及在事件发生时需要通知的run-loop监听器，当被监听的事件发生时，run-loop会产生一个消息，被run-loop监听器补货，从而执行预定的动作。

  

