---
title: php-fpm 配置文件优化
date: 2018-07-11
categories: PHP
tag: phpfpm
---

## PHP-FPM
- CGI是Common Gateway Interface（通用网管协议），用于让交互程序和Web服务器通信的协议。它负责处理URL的请求，启动一个进程，将客户端发送的数据作为输入，由Web服务器收集程序的输出并加上合适的头部，再发送回客户端。
- FastCGI是基于CGI的增强版本的协议，不同于创建新的进程来服务请求，使用持续的进程和创建的子进程来处理一连串的进程，这些进程由FastCGI服务器管理，开销更小，效率更高。
- PHP-FPM是PHP实现的FastCGI Process Manager(FastCGI进程管理器), 用于替换PHP FastCGI的大部分附加功能，适用于高负载网站

常用的PHP-FPM 配置
```bash
[global]  //全局配置
pid = /usr/local/php/var/run/php-fpm.pid  //PID 文件的位置
error_log = /data/logs/php/php-fpm.log  //错误日志的位置
log_level = error //错误级别

[www] //进程池配置.php-fpm可以配置多个进程池来针对不同的项目
listen = /tmp/php-cgi.sock //设置接受 FastCGI 请求的地址。可用格式为：'ip:port'，'port'，'/path/to/unix/socket'。每个进程池都需要设置。
user = www //FPM 进程运行的Unix用户。必须设置。
group = www //FPM 进程运行的 Unix 用户组。如果不设置，就使用默认用户的用户组。
listen.mode = 0666 //默认为运行所使用的用户和组，权限为 066。
pm = dynamic //设置进程管理器如何管理子进程。可用值：static，ondemand，dynamic。必须设置。
pm.max_children = 700  //pm 设置为 static 时表示创建的子进程的数量，pm 设置为 dynamic 时表示最大可创建的子进程的数量。必须设置。
pm.start_servers = 50 //设置启动时创建的子进程数目。仅在 pm 设置为 dynamic 时使用。默认值：min_spare_servers + (max_spare_servers - min_spare_servers) / 2。
pm.min_spare_servers = 50 //设置空闲服务进程的最低数目。仅在 pm 设置为 dynamic 时使用。必须设置。
pm.max_spare_servers = 100 //设置空闲服务进程的最大数目。仅在 pm 设置为 dynamic 时使用。必须设置。
pm.max_requests = 500 //设置每个子进程重生之前服务的请求数
pm.status_path = /phpfpm_status //FPM 状态页面的网址
request_terminate_timeout = 3600 //设置单个请求的超时中止时间。该选项可能会对 php.ini 设置中的 'max_execution_time' 因为某些特殊原因没有中止运行的脚本有用。
request_slowlog_timeout = 30s //当一个请求该设置的超时时间后，就会将对应的 PHP 调用堆栈信息完整写入到慢日志中
slowlog = /data/logs/php/php-fpm.slow.log //慢请求的记录日志
```

[官网PHP-FPM 配置](https://www.php.net/manual/zh/install.fpm.configuration.php)


### pm 配置
`pm` 指的是process manager，指定进程管理器如何控制子进程的数量，它为必填项，支持3个值：

- static: 使用固定的子进程数量，由pm.max_children指定
- dynamic：基于下面的参数动态的调整子进程的数量，至少有一个子进程
    + pm.max_chidren: 可以同时存活的子进程的最大数量
    + pm.start_servers: 启动时创建的子进程数量，默认值为min_spare_servers + (max_spare_servers - min_spare_servers) / 2
    + pm.min_spare_servers: 空闲状态的子进程的最小数量，如果不足，新的子进程会被自动创建
    + pm.max_spare_servers: 空闲状态的子进程的最大数量，如果超过，一些子进程会被杀死
- ondemand: 启动时不会创建子进程，当新的请求到达时才创建。会使用下面两个参数：
    + pm.max_children
    + pm.process_idle_timeout 子进程的空闲超时时间，如果超时时间到没有新的请求可以服务，则会被杀死

pm.max_requests = 1000 每一个子进程服务的请求数量做限制，防止无限制的内存泄漏

> php进程一般来说刚启动时是8M左右，运行一段时间由于内存泄漏和缓存会上涨到30M左右，可以根据这个预期内存大小设定进程的数量
> 对于内存大的服务器来说，指定静态的max_children实际上更为妥当，因为这样不需要进行额外的进程数目控制，会提高效率。因为频繁开关php-fpm进程也会有时滞，所以内存够大的情况下开静态效果会更好。
> 
### nginx与php-fpm通信的两种方式

nginx配置php如下：
```bash
location ~ \.php$ {
    try_files $uri /index.php =404;
    fastcgi_pass 127.0.0.1:9000; //nginx和php-fmp的通信
    fastcgi_index index.php;
    fastcgi_buffers 16 16k;
    fastcgi_buffer_size 32k;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
}
```
fastcgi_pass 有2中方式如下：

1. unix socket listen = /tmp/php-fpm.sock  fastcgi_pass unix:/tmp/php-fpm.sock;
unix socket方式因为不走网络，的确可以提高Nginx和php-fpm通信的性能，但在高并发时会不稳定。
Nginx会频繁报错：connect() to unix:/dev/shm/php-fcgi.sock failed (11: Resource temporarily unavailable) while connecting to upstream
运维大大说可以通过下面两种方式提高稳定性：
    - 调高nginx和php-fpm中的backlog
        + 配置方法为：在nginx配置文件中这个域名的server下，在listen 80后面添加default backlog=1024。
        + 同时配置php-fpm.conf中的listen.backlog为1024，默认为128。
    - 增加sock文件和php-fpm实例数
        再新建一个sock文件，在Nginx中通过upstream模块将请求负载均衡到两个sock文件背后的两套php-fpm实例上。

2. tcp socket  listen = 127.0.0.1:9000  fastcgi_pass 127.0.0.1:9000;
tcp协议能保证数据的正确性，unix sock不能保证。
可以跨服务器，当nginx和php-fpm不在同一台机器上时，只能使用这种方式

### php-fpm慢日志
当一个请求php的处理时间超过15秒后，就会将对应的PHP调用堆栈信息完整写入到慢日志中。
```bash
slowlog = /usr/local/var/log/php-fpm.log.slow
request_slowlog_timeout = 15s
```

### 自动重启主进程

为了避免PHP-FPM主进程由于某些XX的PHP代码挂掉，需要设置重启的全局配置：

```bash
#如果在1min内有10个子进程被中断失效，重启主进程
emergency_restart_threshold = 10
emergency_restart_interval = 1m
```

### backlog

backlog的定义是已连接但未进行accept处理的SOCKET队列大小
backlog的大小FPM的处理能力有关。
backlog太大，导致FPM处理不过来，nginx那边等待超时，断开连接，报504 gateway timeout错。同时FPM处理完准备write 数据给nginx时，发现TCP连接断开了，报“Broken pipe”。
backlog太小，NGINX之类client，根本进入不了FPM的accept queue，报“502 Bad Gateway”错。
所以，这还得去根据FPM的QPS来决定backlog的大小。计算方式最好为QPS=backlog。对了这里的QPS是正常业务下的QPS，千万别用echo hello world这种结果的QPS去欺骗自己。
当然，backlog的数值，如果指定在FPM中的话，记得把操作系统的net.core.somaxconn设置的起码比它大。
另外，ubuntu server 1404上/proc/sys/net/core/somaxconn 跟/proc/sys/net/ipv4/tcp_max_syn_backlog 默认值都是128。
对于测试时，TCP数据包已经drop掉的未进入syns queue，以及未进入accept queue的数据包，可以用netstat -s来查看
