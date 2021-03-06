
### nginx 502 Bad Gateway 错误解决办法

参考：http://jingyan.baidu.com/article/eb9f7b6dacaac3869364e88f.html

一些运行在Nginx上的网站有时候会出现“502 Bad Gateway”错误，有些时候甚至频繁的出现。以下是小编搜集整理的一些Nginx 502错误的排查方法，供参考：

　　Nginx 502错误的原因比较多，是因为在代理模式下后端服务器出现问题引起的。这些错误一般都不是nginx本身的问题，一定要从后端找原因！但nginx把这些出错都揽在自己身上了，着实让nginx的推广者备受置疑，毕竟从字眼上理解，bad gateway？不就是bad nginx吗？让不了解的人看到，会直接把责任推在nginx身上，希望nginx下一个版本会把出错提示写稍微友好一些，至少不会是现在简单的一句 502 Bad Gateway，另外还不忘附上自己的大名。

Nginx 502的触发条件

　　502错误最通常的出现情况就是后端主机当机。在upstream配置里有这么一项配置：proxy_next_upstream，这个配置指定了 nginx在从一个后端主机取数据遇到何种错误时会转到下一个后端主机，里头写上的就是会出现502的所有情况拉，默认是error timeout。error就是当机、断线之类的，timeout就是读取堵塞超时，比较容易理解。我一般是全写上的：

proxy_next_upstream error timeout invalid_header http_500 http_503;　　不过现在可能我要去掉http_500这一项了，http_500指定后端返回500错误时会转一个主机，后端的jsp出错的话，本来会打印一堆 stacktrace的错误信息，现在被502取代了。但公司的程序员可不这么认为，他们认定是nginx出现了错误，我实在没空跟他们解释502的原理 了……

503错误就可以保留，因为后端通常是apache resin，如果apache死机就是error，但resin死机，仅仅是503，所以还是有必要保留的。

解决办法

遇到502问题，可以优先考虑按照以下两个步骤去解决。

1、查看当前的PHP FastCGI进程数是否够用：

复制代码 代码如下:

netstat -anpo | grep "php-cgi" | wc -l

如果实际使用的“FastCGI进程数”接近预设的“FastCGI进程数”，那么，说明“FastCGI进程数”不够用，需要增大。

2、部分PHP程序的执行时间超过了Nginx的等待时间，可以适当增加nginx.conf配置文件中FastCGI的timeout时间，例如：

复制代码 代码如下:

 http  {  fastcgi_connect_timeout 300;  fastcgi_send_timeout 300;  fastcgi_read_timeout 300;  ......  }  ......

php.ini中memory_limit设低了会出错，修改了php.ini的memory_limit为64M，重启nginx，发现好了，原来是PHP的内存不足了。

　　如果这样修改了还解决不了问题，可以参考下面这些方案：

一、max-children和max-requests

　　一台服务器上运行着nginx php(fpm) xcache，访问量日均 300W pv左右。

　　最近经常会出现这样的情况：php页面打开很慢，cpu使用率突然降至很低，系统负载突然升至很高，查看网卡的流量，也会发现突然降到了很低。这种情况只持续数秒钟就恢复了。

　　检查php-fpm的日志文件发现了一些线索。

复制代码 代码如下:

Sep 30 08:32:23.289973 [NOTICE] fpm_unix_init_main(), line 271: getrlimit(nofile): max:51200, cur:51200  Sep 30 08:32:23.290212 [NOTICE] fpm_sockets_init_main(), line 371: using inherited socket fd=10, “127.0.0.1:9000″  Sep 30 08:32:23.290342 [NOTICE] fpm_event_init_main(), line 109: libevent: using epoll  Sep 30 08:32:23.296426 [NOTICE] fpm_init(), line 47: fpm is running, pid 30587　

在这几句的前面，是1000多行的关闭children和开启children的日志。

　　原来，php-fpm有一个参数 max_requests，该参数指明了，每个children最多处理多少个请求后便会被关闭，默认的设置是500。因为php是把请求轮询给每个 children，在大流量下，每个childre到达max_requests所用的时间都差不多，这样就造成所有的children基本上在同一时间 被关闭。

　　在这期间，nginx无法将php文件转交给php-fpm处理，所以cpu会降至很低(不用处理php，更不用执行sql)，而负载会升至很高(关闭和开启children、nginx等待php-fpm)，网卡流量也降至很低(nginx无法生成数据传输给客户端)

　　解决问题很简单，增加children的数量，并且将 max_requests 设置未 0 或者一个比较大的值：

　　打开 /usr/local/php/etc/php-fpm.conf调大以下两个参数(根据服务器实际情况，过大也不行）

复制代码 代码如下:

<value>5120</value><value>600</value>　　

然后重启php-fpm。

二、增加缓冲区容量大小

　　将nginx的error log打开，发现“pstream sent too big header while reading response header from upstream”这样的错误提示。查阅了一下资料，大意是nginx缓冲区有一个bug造成的,我们网站的页面消耗占用缓冲区可能过大。参考老外写的修 改办法增加了缓冲区容量大小设置，502问题彻底解决。后来系统管理员又对参数做了调整只保留了2个设置参数：client head buffer，fastcgi buffer size。

三、request_terminate_timeout

　　如果主要是在一些post或者数据库操作的时候出现502这种情况，而不是在静态页面操作中常见，那么可以查看一下php-fpm.conf设置中的一项：

request_terminate_timeout

这个值是max_execution_time，就是fast-cgi的执行脚本时间。

0s

0s为关闭，就是无限执行下去。（当时装的时候没仔细看就改了一个数字）问题解决了，执行很长时间也不会出错了。优化fastcgi中，还可以改改这个值5s 看看效果。

php-cgi进程数不够用、php执行时间长、或者是php-cgi进程死掉，都会出现502错误。Nginx 502 Bad Gateway错误的解决办法2

今天，我的VPS频繁提示Nginx 502 Bad Gateway错误了，重启了VPS解决之后又出现，很烦。有点想不通，前两天网站达到了1290的访问量都没有出什么问题，怎么这次就出现了502 Bad Gateway？郁闷啊！！！在搜索了很久，终于找到了不少相关的答案，希望修改之后不会再出现这个错误了。唉，既然在网上找了那么久的答案，那当然得把有用的东西记录下，免得我下次再去谷歌~

由于我是采用了LNMP一键安装包 ，出了问题肯定要先到官方论坛去搜索下了，真好，官方有个这样的置顶帖，大家先瞧瞧。

LNMP一键安装包官方的：

第一种原因：目前lnmp一键安装包比较多的问题就是502 Bad Gateway，大部分情况下原因是在安装php前，脚本中某些lib包可能没有安装上，造成php没有编译安装成功。解决办法：可以尝试根据lnmp一键安装包中的脚本手动安装一下，看看是什么错误导致的。

第二种原因：

在php.ini里，eaccelerator配置项一定要放在Zend Optimizer配置之前，否则也可能引起502 Bad Gateway

第三种原因：

在安装好使用过程中出现502问题，一般是因为默认php-cgi进程是5个，可能因为phpcgi进程不够用而造成502，需要修改/usr/local/php/etc/php-fpm.conf 将其中的max_children值适当增加。

第四种原因：

php执行超时，修改/usr/local/php/etc/php.ini 将max_execution_time 改为300

第五种原因：

磁盘空间不足，如mysql日志占用大量空间

第六种原因：

查看php-cgi进程是否在运行

也有网友给出了另外的解决办法：

Nginx 502 Bad Gateway的含义是请求的PHP-CGI已经执行，但是由于某种原因（一般是读取资源的问题）没有执行完毕而导致PHP-CGI进程终止，一般来说Nginx 502 Bad Gateway和php-fpm.conf的设置有关。

php-fpm.conf有两个至关重要的参数，一个是max_children，另一个是request_terminate_timeout，但是这个值不是通用的，而是需要自己计算的。在安装好使用过程中出现502问题，一般是因为默认php-cgi进程是5个，可能因为phpcgi进程不够用而造成502，需要修改/usr/local/php/etc/php-fpm.conf 将其中的max_children值适当增加。

计算的方式如下：

如果你的服务器性能足够好，且宽带资源足够充足，PHP脚本没有系循环或BUG的话你可以直接将 request_terminate_timeout设置成0s。0s的含义是让PHP-CGI一直执行下去而没有时间限制。而如果你做不到这一点，也就 是说你的PHP-CGI可能出现某个BUG，或者你的宽带不够充足或者其他的原因导致你的PHP-CGI假死那么就建议你给 request_terminate_timeout赋一个值，这个值可以根据服务器的性能进行设定。一般来说性能越好你可以设置越高，20分钟-30分 钟都可以。而max_children这个值又是怎么计算出来的呢？这个值原则上是越大越好，php-cgi的进程多了就会处理的很快，排队的请求就会很少。 设置max_children也需要根据服务器的性能进行设定，一般来说一台服务器正常情况下每一个php-cgi所耗费的内存在20M左右。

按照官方的答案，排查了相关的可能，并结合了网友的答案，得出了下面的解决办法。

1、查看php fastcgi的进程数（max_children值）

代码：netstat -anpo | grep “php-cgi” | wc -l

5（假如显示5）

2、查看当前进程

代码：top观察fastcgi进程数,假如使用的进程数等于或高于5个，说明需要增加（根据你机器实际状况而定）

3、调整/usr/local/php/etc/php-fpm.conf 的相关设置

<value name=”max_children”>10</value><value name=”request_terminate_timeout”>60s</value>max_children最多10个进程，按照每个进程20MB内存，最多200MB。request_terminate_timeout执行的时间为60秒，也就是1分钟。

### 官方文档

链接：http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass

### 重写配置

3. squid3 代理服务器、缓存服务器、加速器

```
# Squid中文权威指南
http://home.arcor.de/pangj/squid/index.html

# 前端nginx负载均衡后端多个squid配置
http://www.eduyo.com/server/linux/829.html

# 在ec2上配置squid http代理服务器笔记
http://www.zuola.com/weblog/2012/12/1873.htm

# 双服务器，CentOS系统，使用Squid和Stunnel搭建翻墙代理服务
http://fuweiyi.com/others/2014/05/15/a-Centos-Squid-Stunnel-proxy.html

# Stunnel + Squid + pac文件快速简单自动过滤翻墙
https://www.fourfire.cc/544.html

# 用NGINX 代理翻墙
http://www.linuxbyte.org/yong-nginx-daili-fanqiang.html
```

2. 重写地址需要rewrite，但对于host和请求参数无效

链接：http://blog.cafeneko.info/2010/10/nginx_rewrite_note/

1. 重写服务器localhost:4200 -> localhost，需要proxy_redirect，但对于image,css,script等无效

链接：http://superuser.com/questions/690916/how-to-make-nginx-rewrite-uris-in-http-body-content

     https://gist.github.com/lukemelia/3714b351c7865d091160

     http://w.gdu.me/wiki/sysadmin/nginx_proxy_redirect.html 【中文】

     http://serverfault.com/questions/586586/nginx-redirect-via-proxy-rewrite-and-preserve-url 【无效果】

插件：https://github.com/yaoweibin/ngx_http_substitutions_filter_module

