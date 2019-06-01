---
title: Laravel系统优化,结合swoole
date: 2018-05-14 
categories: PHP
tag: laravel 
---
### 优化 Laravel 网站打开速度
1. 关闭 debug
打开.env 文件，把 debug 设置为 false.barryvdh/laravel-debugbar包一定要放在require-dev,线上就不要载入了
2. 缓存路由和配置
php artisan route:cache。如果路由中有闭包是会报错的,所以路由中就不要添加处理逻辑
php artisan config:cache
3. composer 优化
composer dump-autoload --optimize
4. Laravel 优化命令
php artisan optimize
5. 使用 Laravel 缓存
``` php
$lists = Cache::remember('key', 20, function () {
    return $this->destination->getList();
});
```
6. 使用 PHP7 并开启 OPcache
开启opcache后需要重启 php-fpm哦

7. nginx 开启 gzip 压缩
Nginx 开启 gzip 可以有效减少服务器带宽的消耗，缺点是会增大 CPU 的占用率，但是很多时候 CPU 往往是空闲最多的。
在nginx的配置中添加如下:
``` bash
gzip on;
gzip_min_length 1k;
gzip_buffers 16 64k;
gzip_http_version 1.1;
gzip_comp_level 9;
gzip_types text/plain application/x-javascript application/javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png font/ttf font/otf image/svg+xml;
gzip_vary on;   
```

`GZIP_MIN_LENGTH `  设置允许压缩的页面最小字节数，页面字节数从 header 头中的 Content-Length 中进行获取。默认值是 0，不管页面多大都压缩。建议设置成大于 1k 的字节数，小于 1k 可能会越压越大。 即: gzip_min_length 1024 

响应头中 Content-Encoding 字段是 gzip，表示该网页是经过 gzip 压缩的。

### laravel + swoole
我们这里使用[laravels](https://github.com/hhxsv5/laravel-s)来优化laravel5.5




### 压测
PHP 7.0.11 + Laravel5.5
服务器配置，就一个小型测试项目在跑，装有一个mysql服务。

``` bash
top - 11:30:03 up 396 days, 10:08,  2 users,  load average: 0.14, 5.14, 7.21
Tasks: 315 total,   1 running, 297 sleeping,  16 stopped,   1 zombie
Cpu0  : 16.3%us,  4.3%sy,  0.0%ni, 75.4%id,  2.0%wa,  0.0%hi,  1.3%si,  0.7%st
Cpu1  :  1.7%us,  1.7%sy,  0.0%ni, 96.4%id,  0.0%wa,  0.0%hi,  0.0%si,  0.3%st
Cpu2  :  1.0%us,  2.0%sy,  0.0%ni, 96.7%id,  0.3%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu3  :  1.3%us,  1.3%sy,  0.0%ni, 97.4%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu4  :  1.3%us,  1.3%sy,  0.0%ni, 97.4%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu5  :  9.3%us,  2.7%sy,  0.0%ni, 88.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu6  :  1.0%us,  3.0%sy,  0.0%ni, 96.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu7  :  0.7%us,  2.3%sy,  0.0%ni, 97.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Mem:  16333892k total, 15396272k used,   937620k free,   191568k buffers
Swap:        0k total,        0k used,        0k free, 10030768k cached

[root@tracknumber_share niftyAdmin]# free -m
             total       used       free     shared    buffers     cached
Mem:         15951      15036        914          0        187       9797
-/+ buffers/cache:       5051      10899
Swap:            0          0          0

```
测试都是在本地使用简单的ab测试，目的只是想横向对比 `php-fpm + nginx`  和 `swoole + nginx` 的性能

`./ab -n 5000 -c 500  http://nt.valsun.cn/api/getData`

逻辑很简单,启用laravel5.5的框架通过api路由直接输出,laravel已经使用上面的优化措施
``` php
public function getData()
{
    echo '3123123';
}
```

#### nginx + php-fpm 测试结果

压测时服务器的指标，这个是8核的只是负载17其实还好啦
``` bash
top - 11:41:58 up 396 days, 10:20,  2 users,  load average: 17.20, 12.25, 11.45
```

ab压测结果

``` bash
Server Software:        BWS
Server Hostname:        nt.valsun.cn
Server Port:            80

Document Path:          /api/getData
Document Length:        1436 bytes

Concurrency Level:      1000
Time taken for tests:   10.291 seconds
Complete requests:      5000
Failed requests:        954
   (Connect: 0, Receive: 0, Length: 954, Exceptions: 0)
Non-2xx responses:      5000
Total transferred:      7339154 bytes
HTML transferred:       5967972 bytes
Requests per second:    485.84 [#/sec] (mean)
Time per request:       2058.296 [ms] (mean)
Time per request:       2.058 [ms] (mean, across all concurrent requests)
Transfer rate:          696.42 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    2  42.5      1    3004
Processing:   112 1783 1146.6   1527    3988
Waiting:        8 1579 1098.9   1368    3978
Total:        113 1785 1146.8   1528    3989

Percentage of the requests served within a certain time (ms)
  50%   1528
  66%   1751
  75%   2064
  80%   2434
  90%   3946
  95%   3969
  98%   3977
  99%   3978
 100%   3989 (longest request)
```

#### ngingx + swoole 压测