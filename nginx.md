## 一.nginx的特点

> nginx的特点：
>
> 1.稳定性极强，不间断运行
>
> 2.nginx提供了非常丰富的配置实例
>
> 3.占用内存小，并发能力强

## 二. Nginx的安装

#### 2.1安装nginx

```yml
version: '3.1'
services:
  nginx:
    restart: always
    image: daocloud.io/library/nginx
    container_name: nginx
    ports:
      - 80:80
```

#### 2.2Nginx配置文件

```json
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;
#以上统称为全局块
#woker_processes 的数值越大，nginx的并发能力就越强
#error_log 代表nginx错误日志的地址

#events块
#worker_connections 的数值越大，nginx的并发能力越强
events {
    worker_connections  1024;
}

#http块
#include 代表引入一个外部的文件-》 /etc/nginx/mime.types 中存放大量的媒体类型
#include /etc/nginx/conf.d/*.conf; -> 引入了conf.d目录下的以conf为结尾的配置文件
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}

```

conf.d 下面的配置文件，去掉注释部分，可以替换掉上面的<include /etc/nginx/conf.d/*.conf;>

```json
#server块
#listen:代表nginx监听的端口号
#localhost：代表nginx接收请求的ip

server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;
	#location块
	#root：将接受到的请求根据 /usr/share/nginx/html;去查找静态资源
	#index：默认去上述的路径中找到index.html或者index.htm
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}

```

#### 2.3修改docker-compose.yml文件

```yml
version: '3.1'
services:
  nginx:
    restart: always
    image: daocloud.io/library/nginx
    container_name: nginx
    ports:
      - 80:80
    volumes:
      - /opt/docker_nginx/conf.d/:/etc/nginx/conf.d/
```

## 三.Nginx的反向代理

#### 3.1正向代理

> 正向代理：
>
> 1.正向代理服务器由客户端设立的
>
> 2.客户端了解代理服务器和目标服务器都是谁
>
> 3.帮助实现突破访问权限，提高访问的速度，对目标服务器隐藏客户端的ip地址

![1602235995218](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1602235995218.png)

#### 3.2反向代理

> 反向代理：
>
> 1.反向代理服务器是配置在服务端的
>
> 2.客户端是不知道访问的哪一台服务器
>
> 3.达到负载均衡，并且隐藏服务器的真正ip地址

![1602236111007](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1602236111007.png)

#### 3.3基于Nginx实现反向代理

> 准备一个目标服务器
>
> 启动了之前的tomcat服务器
>
> 编写nginx的配置文件，通过nginx访问到tomcat服务器

```json
server{
  listen 80;
  server_name localhost;
    #基于反向代理访问到tomcat服务器
  location / {
    proxy_pass http://10.77.100.38:8080/;
  }  
}
```

#### 3.4关于Nginx的location路径映射

> 优先级关系
>
> (location =) > (location /xx/yyy) > (location ^~) > (location ~, ~*) > (location /起始路径) > (location /)

```json
#1.=匹配
location = / {
  #精准匹配，主机名后面不能带任何的字符串
}
```

```json
#2.通用匹配
location /xxx {
    #匹配所有以/xxx开头的路径
}
```

```json
#3.正则匹配
location ~ /xx {
  #匹配所有以/xx开头的路径
}
```

```json
#4.匹配开头路径
location ^~ /images/ {
    #匹配所有以/images开头的路径
}
```

```json
#5.~* \.(gif|jpg|png)$ {
#匹配以gif或jpg或png为结尾的路径
}
```

## 四.Nginx负载均衡

#### 4.1Nginx负载均衡的策略

> 1.轮询：
>
>    将客户端发起的请求，平均分配给每一台服务器
>
> 2.权重：
>
>    会将客户端的请求，根据服务器的 权重值不通，分配不同的数量
>
> 3.ip_hash：
>
>    基于发起请求的客户端的ip地址不通，他始终会将请求发送到指定的服务器上

#### 4.2 轮询

> 实现Nginx轮询负载均衡机制，需要在配置文件中提那家以下内容

```json
upstream 自定义名称(不要用下划线) {
    server ip:port;
    server ip:port;
    ...
}
server {
    listen 80;
    server_name localhost;
    location / {
      proxy_pass http://自定义名称/;
    }
}
```

#### 4.3权重策略

```json
upstream 自定义名称(不要用下划线) {
    #weight代表权重，数值大，访问的概率就大
    server ip:port weight=10;
    server ip:port weight=2;
    ...
}
server {
    listen 80;
    server_name localhost;
    location / {
      proxy_pass http://自定义名称/;
    }
}
```

#### 4.4ip_hash

```json
upstream 自定义名称(不要用下划线) {
    #开启ip_hash策略，就和下面的weight没有关系了，只执行ip_hash策略
    ip_hash;
    server ip:port weight=10;
    server ip:port weight=2;
    ...
}
server {
    listen 80;
    server_name localhost;
    location / {
      proxy_pass http://自定义名称/;
    }
}
```

## 五.Nginx动静分离

> Nginx的并发能力公式：
>
>    woker_processes * woker_connections /4 | 2 = Nginx最终的并发能力
>
> 动态资源需要除以 4， 静态资源需要除以2

#### 5.1动态资源代理

```json
#配置如下
location / {
    proxy_pass 路径;
}
```

#### 5.2静态资源代理

```json
#配置如下
location / {
    root 静态资源路径;
    index 默认访问路径下的什么资源;
    autoindex on; #代表展示静态资源全部内容，以列表刑事展开
}

#先修改docker-compose.yml, 添加一个volume 映射nginx中的静态资源
/opt/docker_nginx/img/:/data/img
/opt/docker_nginx/html:/data/html
#在这两个目录下分别放入图片和html文件
#修改nginx conf配置
location /html {
    root /data;
    index index.html;
}

location /img {
    root /data;
    autoindex on;
}
```

## 六.Nginx集群

#### 6.1引言

> 防止nginx单点故障的发生，造成服务不可用。

1.多个nginx服务器

2.每个都安装keepalived程序，来监测nginx是否存活

3.haproxy 来代理客户端的请求，通过keepalived来决定发送到哪个nginx服务器

![1602297252530](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1602297252530.png)

#### 6.2搭建nginx集群

