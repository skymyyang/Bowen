## Vector 处理kafka中日志并转发给loki-02

### 部署minio



minio部署要求：

1. 如果是集群部署的话，所有的节点必须有一块独立的磁盘用于存放minio的数据。不能与与'/'分区或其他分区在同一硬盘上。否则会报错。`Disk `   `http://rnode4:9000/data1/export1` `   ` `is part of root disk, will not be used (*errors.errorString)`
2. 且Linux 内核版本>4.0.0

基于RockyLinux8.5手动部署

1.  系统初始化

   ```shell
   # 关闭防火墙
   $ setenforce 0
   $ sed -ri '/^[^#]*SELINUX=/s#=.+$#=disabled#' /etc/selinux/config
   $ systemctl stop firewalld
   $ systemctl disable firewalld
   # 镜像源
   $ sed -e 's|^mirrorlist=|#mirrorlist=|g' \
       -e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.aliyun.com/rockylinux|g' \
       -i.bak \
       /etc/yum.repos.d/Rocky-*.repo
       
   $ dnf install epel-release -y
   $ sed -i 's|^#baseurl=https://download.example/pub|baseurl=https://mirrors.aliyun.com|' /etc/yum.repos.d/epel*
   $ sed -i 's|^metalink|#metalink|' /etc/yum.repos.d/epel*
   # 时区修改
   $ ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
   # 配置时间同步
   $ dnf install chrony -y #安装chrony，可能默认已安装
   #修改配置文件
   $ vi /etc/chrony.conf
   # 修改pool 2.pool.ntp.org iburst 
   pool ntp.aliyun.com iburst
   # 然后进行启动
   $ systemctl enable chronyd 
   $ systemctl restart chronyd
   $ systemctl status chronyd
   # 配置所有节点的hosts文件
   # 192.168.50.153 minio-node1
   # 192.168.50.154 minio-node2
   # 192.168.50.155 minio-node3
   # 192.168.50.156 minio-node4
   ```

2. ulimit参数优化

   ```
   cat>/etc/security/limits.d/baseopen.conf<<EOF
   *       soft    nproc   131072
   *       hard    nproc   131072
   *       soft    nofile  131072
   *       hard    nofile  131072
   root    soft    nproc   131072
   root    hard    nproc   131072
   root    soft    nofile  131072
   root    hard    nofile  131072
   EOF
   ```

3. sysctl优化

   ```shell
   $ wget https://raw.githubusercontent.com/skymyyang/ansible-minio-cluster-install/main/library/kernel-tuning.sh
   $ chmod +x kernel-tuning.sh
   $ bash kernel-tuning.sh
   $ vim /etc/default/grub
   
   GRUB_CMDLINE_LINUX="crashkernel=auto resume=/dev/mapper/rl-swap rd.lvm.lv=rl/root rd.lvm.lv=rl/swap transparent_hugepage=madvise"
   
   $ grub2-mkconfig -o /boot/grub2/grub.cfg
   $ reboot
   ```

4. 每个节点添加两块磁盘，并进行分区设置

   ```shell
   $ fdisk /dev/sdb
   $ fdisk /dev/sdc
   $ mkfs.xfs /dev/sdb1
   $ mkfs.xfs /dev/sdc1
   $ mkdir /data1
   $ mkdir /data2
   $ mount /dev/sdb1 /data1
   $ mount /dev/sdc1 /data2
   # 查看uuid并设置开启自动挂载
   $ blkid /dev/sdb1
   $ cat /etc/fstab
   ```

5. 下载二进制文件

   ```shell
   $ wget https://dl.min.io/server/minio/release/linux-amd64/minio
   $ mv minio /usr/local/bin/
   $ chmod +x /usr/local/bin/minio
   $ wget https://dl.min.io/client/mc/release/linux-amd64/mc
   $ mv mc /usr/local/bin/
   $ chmod +x /usr/local/bin/mc
   ```

6. 创建用户

   ```shell
   $ groupadd minio
   $ useradd minio -g minio -s /sbin/nologin
   ```

7. 创建目录

   ```shell
   $ mkdir /etc/minio
   $ mkdir /etc/minio/ssl
   $ mkdir /etc/minio/policy
   $ chown -R minio:minio /etc/minio
   ```

8. 创建数据目录

   ```shell
   $ mkdir /data1/export1
   $ mkdir /data2/export1
   $ chown -R minio:minio /data1/export1
   $ chown -R minio:minio /data2/export1
   ```

9. 设置配置文件`/etc/minio/minio.conf`

   ```
   # Minio local/remote volumes.
   MINIO_VOLUMES="http://minio-node{1...4}/data{1...2}/export1"
   
   
   # Minio cli options.
   MINIO_OPTS="--address :9000 --console-address :9001"
   
   # Access Key of the server.
   MINIO_ROOT_USER="minioadmin"
   # Secret key of the server.
   MINIO_ROOT_PASSWORD="minioadmin"
   ```

10. 设置开机启动脚本

    ```shell
    [root@vm153]# cat /etc/systemd/system/minio.service 
    [Unit]
    Description=MinIO
    Documentation=https://docs.min.io
    Wants=network-online.target
    After=network-online.target
    AssertFileIsExecutable=/usr/local/bin/minio
    
    [Service]
    WorkingDirectory=/usr/local/
    
    User=minio
    Group=minio
    #ProtectProc=invisible
    
    EnvironmentFile=/etc/minio/minio.conf
    ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/minio/minio.conf\"; exit 1; fi"
    
    ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
    
    # Let systemd restart this service always
    Restart=always
    
    # Specifies the maximum file descriptor number that can be opened by this process
    LimitNOFILE=65536
    
    # Specifies the maximum number of threads this process can create
    TasksMax=infinity
    
    # Disable timeout logic and wait until process is stopped
    TimeoutStopSec=infinity
    SendSIGKILL=no
    
    
    [Install]
    WantedBy=multi-user.target
    
    $ systemctl enable minio
    $ systemctl restart minio
    ```

11. 配置nginx进行反向代理，有条件的可以增加keepalived进行高可用

    ```shell
    dnf install nginx -y
    ```

12.  nginx配置详情

      nginx.conf

    ```toml
    user nginx;
    worker_processes auto;
    worker_rlimit_nofile 65535;
    error_log /var/log/nginx/error.log;
    pid /run/nginx.pid;
    
    # Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
    include /usr/share/nginx/modules/*.conf;
    
    events {
        multi_accept       on;
        use epoll;
        worker_connections 65535;
    }
    http {
        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';
    
        access_log  off;
    
        charset             utf-8;
        sendfile            on;
        tcp_nopush          on;
        tcp_nodelay         on;
        keepalive_timeout   65;
        types_hash_max_size 2048;
        types_hash_bucket_size 64;
        client_max_body_size   16M;
    
        include             /etc/nginx/mime.types;
        default_type        application/octet-stream;
    
    
    
        # SSL
        ssl_session_timeout    1d;
        ssl_session_cache      shared:SSL:10m;
        ssl_session_tickets    off;
    
        #  Diffie-Hellman parameter for DHE ciphersuites
        ssl_dhparam            /etc/nginx/dhparam.pem;
    
        # Mozilla Intermediate configuration
        ssl_protocols          TLSv1.2 TLSv1.3;
        ssl_ciphers            ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
        # OCSP Stapling
        ssl_stapling           on;
        ssl_stapling_verify    on;
    
        # Load modular configuration files from the /etc/nginx/conf.d directory.
        # See http://nginx.org/en/docs/ngx_core_module.html#include
        # for more information.
        include /etc/nginx/conf.d/*.conf;
    
        server {
            listen       80 default_server;
            listen       [::]:80 default_server;
            server_name  _;
            root         /usr/share/nginx/html;
    
            # Load configuration files for the default server block.
            include /etc/nginx/default.d/*.conf;
    
            location / {
            }
    
            error_page 404 /404.html;
                location = /40x.html {
            }
    
            error_page 500 502 503 504 /50x.html;
                location = /50x.html {
            }
        }
        }
    ```

    general.conf

    ```
    [root@vm153 nginx]# cat default.d/general.conf 
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
    
    # assets, media
    #location ~* \.(?:css(\.map)?|js(\.map)?|jpe?g|png|gif|ico|cur|heic|webp|tiff?|mp3|m4a|aac|ogg|midi?|wav|mp4|mov|webm|mpe?g|avi|ogv|flv|wmv)$ {
    #    expires    7d;
    #    access_log off;
    #}
    
    # svg, fonts
    #location ~* \.(?:svgz?|ttf|ttc|otf|eot|woff2?)$ {
    #    add_header Access-Control-Allow-Origin "*";
    #    expires    7d;
    #    access_log off;
    #}
    
    # gzip
    gzip            on;
    gzip_vary       on;
    gzip_proxied    any;
    gzip_comp_level 6;
    gzip_types      text/plain text/css text/xml application/json application/javascript application/rss+xml application/atom+xml image/svg+xml;
    ```

    s3主要配置文件:

    ```
    [root@vm153 nginx]# cat conf.d/s3-minio-dev.example.com.conf 
    upstream minio {
            server minio-node1:9000;
            server minio-node2:9000;
            server minio-node3:9000;
            server minio-node4:9000;
        }
    
    upstream console {
            ip_hash;
            server minio-node1:9001;
            server minio-node2:9001;
            server minio-node3:9001;
            server minio-node4:9001;
        }
    
        server {
            listen       443 ssl http2;
            server_name  s3-min-dev.example.com;
    	# SSL
            ssl_certificate     /etc/nginx/ssl/example.com.pem;
            ssl_certificate_key /etc/nginx/ssl/example.com.key;
    
            # To allow special characters in headers
            ignore_invalid_headers off;
            # Allow any size file to be uploaded.
            # Set to a value such as 1000m; to restrict file size to a specific value
            client_max_body_size 0;
            # To disable buffering
            proxy_buffering off;
            proxy_request_buffering off;
    
            location / {
                proxy_set_header Host $http_host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
    
                proxy_connect_timeout 300;
                # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
                proxy_http_version 1.1;
                proxy_set_header Connection "";
                chunked_transfer_encoding off;
    
                proxy_pass http://minio;
            }
    	# additional config
    	include default.d/general.conf;
        }
    
        server {
            listen       443 ssl http2;
            server_name  s3-console-dev.example.com;
    
    	# SSL
    	ssl_certificate     /etc/nginx/ssl/example.com.pem;
            ssl_certificate_key /etc/nginx/ssl/example.com.key;
    
            # To allow special characters in headers
            ignore_invalid_headers off;
            # Allow any size file to be uploaded.
            # Set to a value such as 1000m; to restrict file size to a specific value
            client_max_body_size 0;
            # To disable buffering
            proxy_buffering off;
            proxy_request_buffering off;
    
            location / {
                proxy_set_header Host $http_host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_set_header X-NginX-Proxy true;
    
                # This is necessary to pass the correct IP to be hashed
                real_ip_header X-Real-IP;
    
                proxy_connect_timeout 300;
    
                # To support websocket
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
    
                chunked_transfer_encoding off;
    
                proxy_pass http://console;
            }
    	# additional config
    	include default.d/general.conf;
        }
    
    ```

13. 启动nginx

    ```shell
    $ systemctl restart nginx
    $ systemctl enable nginx
    ```

    

#### ansible 自动化部署

参考项目：https://github.com/skymyyang/ansible-minio-cluster-install

```shell
$ ansible-galaxy install atosatto.minio
# 使用此命令进行初始化roles；此时会生成/root/.ansible/roles目录，然后删除rm -rf atosatto.minio即可
```

### 基于minio部署loki

