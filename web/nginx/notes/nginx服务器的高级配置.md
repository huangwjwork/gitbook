# nginx服务器的高级配置

```shell
/etc/sysctl.conf
```



## 针对IPv4的内核优化

### net.core.netdev_max_backlog 

队列数据包最大数目，表示每个网络接口接收数据报的速率比内核处理这些包的速率快时，**允许发送到队列的数据包的最大数目**

系统默认值128，NGINX服务器中定义的NGX_LISTEN_BACKLOG默认为511，需要调整参数

```shell
net.core.netdev_max_backlog = 262144
```

### net.core.somaxconn 

系统同时发起的TCP连接数，默认值128，可能导致链接超时或者重传问题，可以根据实际需要结合并发请求书来调节

```shell
net.core.somaxconn = 262144
```

### net.ipv4.tcp_max_orphans 

设置系统中最多允许存在多少TCP套接字不被关联到任何一个用户文件句柄，超过这个数字，米有与用户文件句柄关联的TCP套接字将被立即复位，同时给出警告信息

```shell
net.ipv4.tcp_max_orphans = 262144
```

### net.ipv4.tcp_max_syn_backlog

未收到客户端确认信息的连接请求的最大值。大于128MB内存的系统默认值为1024，小内存则是128。可以增大该参数：

```shell
net.ipv4.tcp_max_syn_backlog = 262144
```

### net.ipv4.tcp_timestamps

用于设置时间戳，避免序列号的卷绕。赋值为0时表示禁止TCP时间戳的支持。

```shell
net.ipv4.tcp_timestamps = 0
```

### net.ipv4.tcp_synack_retries

用于内核放弃TCP连接之前向客户端发送SYN+ACK包的数量。服务器与客户端要通过三次握手建立连接，在第二次握手期间，内核需要发送SYN并附带一个回应前一个SYN的ACK，主要影响这里，建议赋值为1，即内核放弃连接之前发送一次SYN+ACK包：

```shell
net.ipv4.tcp_synack_retries = 1
```

### net.ipv4.tcp_syn_retries

与net.ipv4.tcp_synack_retries类似，设置内核放弃建立连接之前发送SYN包的数量，建议1

```shell 
net.ipv4.tcp_syn_retries = 1
```

## 针对CPU的配置优化

### worker proceses 

用来设置nginx服务的进程数，一般设置为CPU的倍数，我这里配置worker process auto，一个4个CPU线程，会起4个nginx进程

```shell
[root@test nginx]# ps -ef |  grep nginx |  grep -v grep 
root      1425     1  0 May22 ?        00:00:00 nginx: master process nginx
elk       3495  1425  0 11:33 ?        00:00:00 nginx: worker process
elk       3496  1425  0 11:33 ?        00:00:00 nginx: worker process
elk       3497  1425  0 11:33 ?        00:00:00 nginx: worker process
elk       3498  1425  0 11:33 ?        00:00:00 nginx: worker process
[root@test nginx]# cat /proc/cpuinfo  |  grep processor 
processor       : 0
processor       : 1
processor       : 2
processor       : 3
```



### worker_cpu_affinity 

指定每一个线程的启动在哪个CPU核心，用二进制的每一位表示每个CPU核心，如果线程使用某个CPU核心，则对应核心的位置会置为1，不使用则置为0

比如四CPU，起了4个nginx进程，那么需要4位而至今来表示CPU（4321，自右向左）

* 线程1使用第1个CPU核心：0001
* 线程2使用第2个CPU核心：0010
* 线程3使用第3个CPU核心：0100
* 线程4使用第4个CPU核心：1000

如果线程1使用4个核心，那么表示为1111

```shell
worker_cpu_affinity 0001 0010 0100 1000;
```

如果8核心，则需要8位二进制表示CPU 00000001 00000010 10000000

## 与网络连接相关的配置

### 基础：长连接和短连接

http建立tcp连接

短连接：每一个请求都需要建立一次连接

长链接：一段时间内同一客户端只需要监理一次连接

### keepalive_timeout

设置nginx服务器与客户端保持连接的超时时间，可以设置两个时间：第一个时间字段为服务器断开连接的时间，第二个时间字段可选，为客户端主动断开连接的时间，keepalive消息头会发送给客户端）

```shell
keepalive_timeout 60 50;
```

上面的表示60S后服务器与客户端断开连接，客户端在50S后会主动断开与服务器的连接（不会等待服务器的60S）

### send_timeout

建立连接后，会话活动中服务器等待客户端的响应时间。

```shell
send_timeout 10s;
```

代表在连接建立后，***如果10秒内nginx服务器没有收到客户端新的响应***，服务器会自动关闭连接

### client_header_buffer_size

设置nginx服务器允许的客户端请求头的缓冲区大小，默认1KB，可以根据分系统分页大小来设置。

获取系统分页大小：

```shell
[huangwj@instance-1 ~]$ getconf PAGESIZE
4096
```

nginx服务器返回400错误时大概率是由于请求头过大造成，根据系统分页大小，可以设置client_header_buffer_size为4096

```shell
client_header_buffer_size 4k;
```

### multi_accept

配置nginx服务器是否可以尽可能多的接收客户端的网络连接请求，默认off。

## 与事件驱动模型相关的配置

### use

指定nginx服务器使用的事件驱动模型

### work_connections

设置nginx服务每个工作进程允许同时连接客户端的最大数量

```shell
worker_connections 65535；
```

worker_connections受限于内核参数open_file_resource_limit（进程可以打开文件句柄的数量）

查看file-max

```shell
[huangwj@instance-1 ~]$ cat /proc/sys/fs/file-max            
2000000
```

修改file-max

```shell
[huangwj@instance-1 ~]$ cat /etc/sysctl.conf 
fs.file-max = 2000000

[huangwj@instance-1 ~]$ sudo sysctl -p 
fs.file-max = 2000000
```

### worker_rlimit_sigpending

设置linux平台时间信号队列的长度上限

主要影响rtsig事件驱动模型可以保存的最大信号数。nginx每个worker process有自己的时间信号队列用于暂存客户端请求发生信号，超过长度上限后悔自动专用poll模型处理为处理的客户端请求，根据实际的客户端并发请求数量和服务器运行环境的处理能力自行设定：

```shell
worker_rlimit_sigpending 1024;
```

### devpoll_changes和devpoll_enents

用于设置在/dev/poll事件驱动模式下nginx服务器可以与内核之间传递事件的数量

```shell
# 传递给内核的事件数量
devpoll_changes number
# 从内核获取的事件数量
devpoll_events
```

### kqueue_changes和kqueue_events

用于设置在kqueue事件驱动模式下nginx服务器可以与内核之间传递事件的数量

```shell
# 传递给内核的事件数量
kqueue_changes number
# 从内核获取的事件数量
kqueue_events number
```

### epoll_events

设置在epoll事件驱动模式下nginx服务器可以与内核之间传递事件的数量

```shell
epoll_changes number
```

### RTSIG_SIGNO

用于设置rtsig模式使用得当两个信号中的第一个，第二个信号实在第一个信号的编号上+1

```shell
rtsig_signo signo
```

默认的第一个信号设置为SIGRTMIN+10

查看系统支持的SIGRTMIN

```shell
[huangwj@instance-1 ~]$ kill -l  | grep SIGRTMIN
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
```

### rtsig_overflow_events 、rrtsig_overflow_test、rtsig_overflow_threshold

控制rtsig模式中信号队列溢出时nginx服务器的处理方式

rtsig_overflow_events指令指定队列一出事使用poll库处理的事件数，默认16

rtsig_overflow_test指定poll库处理完第几间时间后将清空rtsig模型是用的信号队列，默认32

rtsig_overfolw_threshold指定rtsig模式使用的信号队列中的事件超过多少事就需要清空队列





