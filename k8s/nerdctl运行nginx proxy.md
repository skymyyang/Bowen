生成nginx proxy 配置文件

```bash
$ mkdir -p /etc/kubernetes
$ cat > /etc/kubernetes/nginx.conf << EOF
error_log stderr notice;

worker_processes 2;
worker_rlimit_nofile 130048;
worker_shutdown_timeout 10s;

events {
  multi_accept on;
  use epoll;
  worker_connections 16384;
}

stream {
  upstream kube_apiserver {
    least_conn;
    server apiserver01.k8s.local:6443 max_fails=3 fail_timeout=10s;
    server apiserver02.k8s.local:6443 max_fails=3 fail_timeout=10s;
    server apiserver03.k8s.local:6443 max_fails=3 fail_timeout=10s;
    }

  server {
    listen        8443;
    proxy_pass    kube_apiserver;
    proxy_timeout 10m;
    proxy_connect_timeout 1s;
  }
}

http {
  aio threads;
  aio_write on;
  tcp_nopush on;
  tcp_nodelay on;

  keepalive_timeout 75s;
  keepalive_requests 100;
  reset_timedout_connection on;
  server_tokens off;
  autoindex off;

  server {
    listen 8081;
    location /healthz {
      access_log off;
      return 200;
    }
    location /stub_status {
      stub_status on;
      access_log off;
    }
  }
}
EOF
```

使用容器运行nginx local proxy

```bash
$ nerdctl run --restart=always \
    -v /etc/kubernetes/nginx.conf:/etc/nginx/nginx.conf \
    -v /etc/localtime:/etc/localtime:ro \
    --name kube-nginx \
    --net host \
    -d \
    nginx:alpine
```

验证

```bash
$ nerdctl ps
CONTAINER ID    IMAGE                             COMMAND                   CREATED           STATUS    PORTS    NAMES
ee0b4b4e2209    docker.io/library/nginx:alpine    "/docker-entrypoint.…"    19 seconds ago    Up                 kube-nginx
```

