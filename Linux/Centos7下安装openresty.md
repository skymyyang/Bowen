## CentOS系统下安装openresty

### 添加仓库

你可以在你的 CentOS 系统中添加 `openresty` 仓库，这样就可以便于未来安装或更新我们的软件包（通过 `yum check-update` 命令）。 运行下面的命令就可以添加我们的仓库（对于 CentOS 8 或以上版本，应将下面的 `yum` 都替换成 `dnf`）：

```
wget https://openresty.org/package/centos/openresty.repo
sudo mv openresty.repo /etc/yum.repos.d/

# update the yum index:
sudo yum check-update
```

### 安装

然后就可以像下面这样安装软件包，比如 `openresty`：

```bash
sudo yum install -y openresty
```

如果你想安装命令行工具 `resty`，那么可以像下面这样安装 `openresty-resty` 包：

```bash
sudo yum install -y openresty-resty
```

命令行工具 `opm` 在 `openresty-opm` 包里，而 `restydoc` 工具在 `openresty-doc` 包里头。

列出所有 `openresty` 仓库里头的软件包：

```bash
sudo yum --disablerepo="*" --enablerepo="openresty" list available
```

参考 [OpenResty RPM 包](http://openresty.org/cn/rpm-packages.html)页面获取这些包更多的细节。

对于 CentOS 8 及更新版本，我们只需要将上面的 `yum` 命令都替换成 `dnf` 即可。

### 配置文件优化

该配置文件基于工具[nginxconfig.io](https://github.com/digitalocean/nginxconfig.io)生成GitHub地址是：`https://github.com/digitalocean/nginxconfig.io`



在您的服务器上运行此命令生成**Diffie-Hellman keys**:

```bash
openssl dhparam -out /usr/local/openresty/nginx/conf/dhparam.pem 2048
```



nginx主配置文件

```nginx
user  nobody;
worker_processes  auto;
worker_rlimit_nofile 65535;


pid        logs/nginx.pid;


events {
    use epoll;
    multi_accept       on;
    worker_connections  65535;
}


http {
    charset                utf-8;
    sendfile               on;
    tcp_nopush             on;
    tcp_nodelay            on;
    server_tokens          off;
    log_not_found          off;
    types_hash_max_size    2048;
    types_hash_bucket_size 64;
    client_max_body_size   16M;

    # MIME
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;
    error_log   logs/error.log warn;
   
    #gzip
    gzip  on;
    gzip_vary       on;
    gzip_proxied    any;
    gzip_comp_level 6;
    gzip_types      text/plain text/css text/xml application/json application/javascript application/rss+xml application/atom+xml image/svg+xml;

    # SSL
    ssl_session_timeout    1d;
    ssl_session_cache      shared:SSL:10m;
    ssl_session_tickets    off;

    # Diffie-Hellman parameter for DHE ciphersuites
    ssl_dhparam            /usr/local/openresty/nginx/conf/dhparam.pem;

    # Mozilla Intermediate configuration
    ssl_protocols          TLSv1.2 TLSv1.3;
    ssl_ciphers            ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    # OCSP Stapling
    ssl_stapling           on;
    ssl_stapling_verify    on;
    # Connection header for WebSocket reverse proxy
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ""      close;
    }
    # default
    server {
        listen 80 default;
        server_name _;
        return 500;
    }
    

    # Load configs
    include                /usr/local/openresty/nginx/conf/vhosts/*.conf;

}
```



基于openresty创建对应的目录

```bash
#SSL证书存放目录certs
mkdir /usr/local/openresty/nginx/conf/certs
#nginxconfig.io各种代理以及全局优化配置目录
mkdir /usr/local/openresty/nginx/conf/nginxconfig.io
#vhosts，存放虚拟机主机的配置目录
mkdir /usr/local/openresty/nginx/conf/vhosts
```

`/usr/local/openresty/nginx/conf/nginxconfig.io/proxy.conf`反向代理全局优化

```nginx
#proxy socket
proxy_http_version                 1.1;
proxy_cache_bypass                 $http_upgrade;

# Proxy headers
proxy_set_header Upgrade           $http_upgrade;
proxy_set_header Connection        $connection_upgrade;
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;
proxy_set_header X-Forwarded-Port  $server_port;

# Proxy timeouts
proxy_connect_timeout              300s;
proxy_send_timeout                 300s;
proxy_read_timeout                 300s;
```

`/usr/local/openresty/nginx/conf/nginxconfig.io/general.conf`全局优化配置

```nginx
# favicon.ico
location = /favicon.ico {
    log_not_found off;
    access_log    off;
}

# robots.txt
location = /robots.txt {
    log_not_found off;
    access_log    off;
}
```

`/usr/local/openresty/nginx/conf/nginxconfig.io/security.conf` 安全的优化

```nginx
# security headers
add_header X-XSS-Protection          "1; mode=block" always;
add_header X-Content-Type-Options    "nosniff" always;
add_header Referrer-Policy           "no-referrer-when-downgrade" always;
add_header Content-Security-Policy   "default-src 'self' http: https: ws: wss: data: blob: 'unsafe-inline'; frame-ancestors 'self';" always;
add_header Permissions-Policy        "interest-cohort=()" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# . files
location ~ /\.(?!well-known) {
    deny all;
}
```

### 虚拟主机配置示例

```nginx
server{
    listen       80;
    server_name  www.example.com;
    location / {
      return 301 https://www.example.com$request_uri;
    }
}

server{
    listen       443 ssl http2;
    server_name  www.example.com;
    root         /var/www/example-offical-web;
    
    #SSL
    ssl_certificate         certs/STAR_example_com_integrated.crt;
    ssl_certificate_key     certs/STAR_example_com.key;
    
    # index.html fallback
    location / {
        try_files $uri $uri/ /index.html;
    }
    
    location /api/ {
        proxy_pass http://127.0.0.1:8080/;
        include    nginxconfig.io/proxy.conf;
    }

    # security
    include             nginxconfig.io/security.conf;

    # additional config
    include nginxconfig.io/general.conf;
    
}
```

