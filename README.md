# HTTPS请求内网穿透，通过Nginx搭配Frp实现。
## 需求：外网发起HTTPS请求，映射到内网网络。
## 实现逻辑
1. 配置 frp服务端、配置 frp客户端，实现 80 端口内网穿透。
2. 服务器 nginx 接收 80 请求，并端口转发给 frp服务器。
3. 服务器 nginx 配置 ssl ，实现 https 请求。

### 第一步：配置 Frp 服务端和客户端
>服务端配置如下
```ini
[common]
bind_udp_port = 7001 ;udp绑定端口7001
bind_addr = 0.0.0.0 ;监听全部ip
bind_port = 7000 ;tcp绑定端口7000
kcp_bind_port = 7000 ;kcp绑定端口7000
```
> 服务器防火墙记得给7000、7001开白名单

> 客户端配置如下
```ini
[common] ;远程服务器配置
server_addr = 138.0.0.1 ;服务器ip
server_port = 7000 ;服务器绑定的tcp端口

[test1] ;穿透配置组
type = tcp ;数据类型
local_port = 80 ;本地端口
remote_port = 7008 ;服务器远程映射端口
```
>测试：访问 http://138.0.0.1:7008 能访问到本地，就是内网穿透好了。

## 第二步：Nginx 接收 80 请求，并端口转发给 frp服务器
> nginx 配置如下
```nginx
server
{
    listen 80; 
    server_name frp.abc.com; #外部域名
    location / {
        proxy_pass http://127.0.0.1:7008; #转发端口就是服务器远程映射的端口
    }
}
```
>测试：访问 http://frp.abc.com 能访问到本地，就是端口成功转发了。

## 第三步：Nginx 配置ssl，实现https请求。
> nginx 配置如下
```nginx
server
{
    listen 80; 
    listen 443 ssl http2;
    server_name frp.abc.com; #外部域名
    
    #SSL-START
    ssl_certificate    /www//cert/fullchain.pem;
    ssl_certificate_key    /www/certprivkey.pem;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    error_page 497  https://$host$request_uri;
    #SSL-END

    location / {
        proxy_pass http://127.0.0.1:7008; #转发端口就是服务器远程映射的端口
    }
}
```
>测试：访问 https://frp.abc.com 能访问到本地，就是https也解决了。
