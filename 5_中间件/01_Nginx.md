# Nginx简介

nginx是一个高性能的web服务器和反向代理服务器

## 代理与反向代理

在互联网中

* 代理，代替客户端访问服务端

    客户端无法直接访问到源服务器

* 反向代理，代替服务端接受客户端的请求

    客户端仍然可以直接访问到源服务器

## 安装和目录结构

安装：

~~~bash
# 安装依赖
yum install gcc openssl openssl-devel pcre pcre-devel zlib zlib-devel -y
# 解压安装包
tar -zxvf nginx-1.22.0.tar.gz
# nginx主目录nginx-1.22.0下执行命令
./configure 
# 编译
make
# 安装
make install

# 开启服务 在/usr/local/nginx/sbin下
./nginx
# 重启服务
./nginx -s reload
~~~

目录结构：

* conf ：配置
* html ：静态资源
* logs ：日志
* sbin ：nginx程序执行文件夹

# nginx核心配置文件

~~~nginx
#配置worker进程运行用户
user  nobody;
#配置工作进程数目，通常等于CPU核心数量的2倍或1倍
worker_processes  1;
#配置全局错误日志及类型 默认是error
error_log  logs/error.log;
error_log  logs/error.log  notice;
error_log  logs/error.log  info;
# 配置进程pid
pid        logs/nginx.pid;

# 配置工作模式和每个工作进程的最大连接数
events {
    # 每个worker进程的连接数上限
    worker_connections  1024;
}

# 配置http服务器，提供反向代理和均衡负载
http {
    #配置nginx支持的http多媒体类型 在mime.types中
    include       mime.types;
    #如果多媒体类型没有在include中，那么默认用二进制流来处理application/octet-stream
    default_type  application/octet-stream;
	#配置日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
	#配置access.log日志已经存放路径和使用的格式
    access_log  logs/access.log  main;
	#开启高效文件传输模式
    sendfile        on;
    #防止网络阻塞
    tcp_nopush     on;
	#长链接超时时间
    #keepalive_timeout  0;
    keepalive_timeout  65;
	# 开启http编码传输
    gzip  on;
	# 配置虚拟主机 可以有多个server
    server {
        #配置监听端口
        listen       80;
        #配置服务名
        server_name  localhost;
		#配置字符集
        charset koi8-r;
		#配置本虚拟主机的访问日志
        access_log  logs/host.access.log  main;
		#配置请求拦截 类似于路径映射，将URI映射到本机的文件 location可以有多个
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
}
~~~

## 静态网站部署

Nginx是一个HTTP的web服务器，可以提供静态服务器的功能

~~~nginx
# location后表示拦截的URI路径
location /image{
    #root表示本地文件入口
    root /opt/static;
    #index指定该入口下默认访问的文件
    index 123.jpg;
}
~~~

上面配置将URI` http://ip:port/image`  映射到 本地资源 `/opt/static/image`

也就是说root映射的是web服务器域名的的根路径

URI域名后的路径，会直接映射到本地资源路径，例如：

~~~
http://ip:port/image/path1/path2/test.png
该路径会访问服务器资源：
/opt/static/image/path1/path2/test.png
~~~

location拦截路径有多种匹配方式，它们有优先级按照优先级从高到底

* 精确匹配 `location =` 
* 最长字符串匹配且完全匹配 `location 完整路径`
* 非正则匹配`location ^~ 路径`
* 正则匹配`location ~,~* 正则顺序`
* 最长字符串匹配，不完全匹配`location 部分起始路径`
* location通配 `location /`

## 反向代理

* 在配置文件http模块下添加upstream模块

    ~~~nginx
    upstream serverNginx {
        # 要被代理的服务器地址
        [ip_hash |least_conn ]  ;
        server 127.0.0.1:8080;
        server 127.0.0.1:8081;
    }
    ~~~

* 用location模块配置代理的拦截路径

    ~~~nginx
    location /web {
        proxy_pass http://serverNginx;
    }
    ~~~

被`/web`拦截的请求将被nginx转发到server_nginx下定义的服务器

### 均衡负载

* 轮询(默认)
* IP绑定 ip_hash
* 最小连接 least_conn 



