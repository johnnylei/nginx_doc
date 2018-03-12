- user user usergroup　# 运行的用户
- worker_process 4/auto # 进程数, auto代表自动匹配最佳进程数，通常进程数与cpu数量设置相等
- worker_cpu_affinity 00000001 00000010 00000100 00001000 #将worker进程与cpu绑定，优化运行效率, cpu亲核性,对效率有显著的提升;如果worker_process auto, 那么这个参数不能再配了
- worker_rlimit_nofile 1024;# 表示一个nginx进程打开最多文件描述符的数量,最好与ulimit -n 数量保持一致
- events
  - use epoll; #使用epoll模式
  - worker_connections 10240; #每个进程最大并发连接数
  - multi_accept on; # 尽可能多的接受请求
- keepalive_timeout 60; # 指定连接生存时间，在最后一个数据包完成之后多久关闭连接,避免反复与服务器建立连接,比较关键;
- gzip on; #开启压缩，这个很关键
  - gzip_min_length 1k; # 小于1k的不压缩
  - gzip_buffer 416k;
  - gzip_http_version 1.1;
  - gzip_comp_level 2; # 压缩级别，级别最大为9, 最小2,越高压缩之后的数据越小,企业一般设置为6;同时值越大对服务器cpu的消耗也就越大
- proxy_connect_timeout 90;# 与被代理服务器建立连接的时间，发起握手后等待响应的时间
- proxy_read_timeout 90; # 连接建立以后，等待后端服务器响应的时间
- proxy_send_timeout 90; # 后端服务器数据回传时间，在规定的时间内后端服务器必须回传所有数据
- expires 30d; #设置http缓存过期时间,一般图片或者静态资源设置30天

## nginx 动静分离
### 动静分离两种方式
- 将静态资源放在独立的服务器上
- 通过nginx配置实现动静分离
```
 // 动态资源
location ~ [^/]\.php(/|$) {
    fastcgi_pass 127.0.0.1:9001;
    #fastcgi_pass unix:/var/run/php-fpm.sock;
    fastcgi_index index.php;
    include fastcgi.conf;
}
// 静态资源
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|flv|ico)$ {
      expires 30d;
      access_log off;
  }   
location ~ .*\.(js|css)?$ {
    expires 7d;
    access_log off;
}
```

## location
```
语法规则： location [=|~|~*|^~] /uri/ { … }
= 开头表示精确匹配
^~ 开头表示uri以某个常规字符串开头，理解为匹配 url路径即可。nginx不对url做编码，因此请求为/static/20%/aa，可以被规则^~ /static/ /aa匹配到（注意是空格）。
~ 开头表示区分大小写的正则匹配
~*  开头表示不区分大小写的正则匹配
!~和!~*分别为区分大小写不匹配及不区分大小写不匹配 的正则
/ 通用匹配，任何请求都会匹配到。
多个location配置的情况下匹配顺序为（参考资料而来，还未实际验证，试试就知道了，不必拘泥，仅供参考）：
首先匹配 =，其次匹配^~, 其次是按文件中顺序的正则匹配，最后是交给 / 通用匹配。当有匹配成功时候，停止匹配，按当前匹配规则处理请求。
```

## 负载均衡
```
http {
: upstream myproject {
: server 127.0.0.1:8000 weight=3;
: server 127.0.0.1:8001;
: server 127.0.0.1:8002;
: server 127.0.0.1:8003;
: }

: server {
: listen 80;
: server_name www.domain.com;
: location / {
: proxy_pass http://myproject;
: }
: }
}
```

## 防盗链
```
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|flv|ico)$ {
    valid_referers none blocked *.51wom.local;
    if ($invalid_referer) {
        return 404;
    }
}
```

## https
It supports checking client certificates with two limitations:
- it is not possible to assign the list of the abolished(废除的) certificates (revocation lists)
- if you have a chain certificate file (sometimes called an intermediate(中间的，中级的;) certificate) you don't specify it separately like you do in Apache. Instead you need to add the information from the chain cert to the end of your main certificate file. This can be done by typing "cat chain.crt >> mysite.com.crt" on the command line. Once that is done you won't use the chain cert file for anything else, you just point Nginx to the main certificate file.
```
worker_processes 1;
http {

  server {
    listen               443;
    ssl                  on;
    ssl_certificate      /usr/local/nginx/conf/cert.pem;
    ssl_certificate_key  /usr/local/nginx/conf/cert.key;
    keepalive_timeout    70;
  }

}
```
### Generate Certificates
```
$ cd /usr/local/nginx/conf
$ openssl genrsa -des3 -out server.key 1024
$ openssl req -new -key server.key -out server.csr
$ cp server.key server.key.org
$ openssl rsa -in server.key.org -out server.key
$ openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```
### Configure the new certificate into nginx.conf:
```
server {

    server_name YOUR_DOMAINNAME_HERE;
    listen 443;
    ssl on;
    ssl_certificate /usr/local/nginx/conf/server.crt;
    ssl_certificate_key /usr/local/nginx/conf/server.key;

}
```
## HttpUpstream模块
这个模块提供一个简单方法来实现在轮询和客户端IP之间的后端服务器负荷平衡。
```
upstream backend  {
  server backend1.example.com weight=5;
  server backend2.example.com:8080;
  server unix:/tmp/backend3;
}

server {
  location / {
    proxy_pass  http://backend;
  }
}
```
### ip_hash
This directive causes requests to be distributed between upstreams based on the IP-address of the client. The key for the hash is the class-C network address of the client. This method guarantees that the client request will always be transferred to the same server. But if this server is considered inoperative, then the request of this client will be transferred to another server. This gives a high probability clients will always connect to the same server.
```
upstream backend {
  ip_hash;
  server   backend1.example.com;
  server   backend2.example.com;
  server   backend3.example.com  down;
  server   backend4.example.com;
}
```
## HttpAccess模块
此模块提供了一个简易的基于主机的访问控制.
```
location / {
: deny    192.168.1.1;
: allow   192.168.1.0/24;
: allow   10.1.1.0/16;
: deny    all;
}
```
在上面的例子中,仅允许网段 10.1.1.0/16 和 192.168.1.0/24中除 192.168.1.1之外的ip访问.

当执行很多规则时,最好使用 ngx_http_geo_module 模块.
## HttpAuthBasic模块
该模块可以使你使用用户名和密码基于 HTTP 基本认证方法来保护你的站点或其部分内容。
```
location  /  {
: auth_basic            "Restricted";
: auth_basic_user_file  conf/htpasswd;
}
```
## HttpHeaders模块
void (
  *signal(
    int sig,
    void (*func)(int)
  )
)(int);
