- user user usergroup　# 运行的用户
- worker_process 4/auto # 进程数, auto代表自动匹配最佳进程数，通常进程数与cpu数量设置相等
- worker_process_affinity 00000001 00000010 00000100 00001000 #将worker进程与cpu绑定，优化运行效率, cpu亲核性,对效率有显著的提升;如果worker_process auto, 那么这个参数不能再配了
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
