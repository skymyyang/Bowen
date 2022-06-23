## 基于docker执行器构建JAVA流水线

前面我们实现了基于docker容器，实现了前端的项目发布。这里我们进行后端JAVA构建的实现。



项目代码：[`https://github.com/skymyyang/springboot-helloworld`](https://github.com/skymyyang/springboot-helloworld)



## 优化docker执行器



```toml
[root@rnode1 springboot-helloworld]# cat /etc/gitlab-runner/config.toml 
concurrent = 10  #定义并行任务
check_interval = 0
log_level = "warning"

[session_server]
  session_timeout = 1800

[[runners]]
  name = "rnode1.example.com"
  url = "http://192.168.10.89/"
  token = "xbHNKky2KnqsoMZEtPiB"
  executor = "shell"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]

[[runners]]
  name = "rnode1-docker"
  url = "http://192.168.10.89/"
  token = "o2WzFwHZy-hiEwCH1WUJ"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "iqimei/alpine:v3.15-ssh-git" #自定义基础镜像
    #最好自己构建一个help镜像，避免因无法拉取gitlab官网镜像，导致构建失败
    helper_image = "iqimei/gitlab-runner-helper:x86_64-febb2a09"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    #volume进行了缓存持久化
    #然后将m2仓库依赖进行持久化，避免重复下载导致构建过慢
    #挂载docker的sock文件，在进行docker in docker 构建镜像时，不再依赖docker服务
    #挂载宿主机的docker配置文件，进行镜像拉取加速
    #共享宿主机的hosts文件，可避免私服DNS无法解析，导致无法正常推送
    volumes = ["/data/cache:/cache", "/data/gitlab-runner-home/m2:/root/.m2", "/var/run/docker.sock:/var/run/docker.sock:ro", "/etc/docker/daemon.json:/etc/docker/daemon.json:ro","/etc/hosts:/etc/hosts:ro"]
    pull_policy = ["if-not-present"]  #优化镜像拉取策略，如果宿主机存在镜像，则不再进行拉取
    shm_size = 0
```



关于docker in docker 镜像构建参考链接：[`https://docs.gitlab.com/runner/executors/docker.html`](https://docs.gitlab.com/runner/executors/docker.html)



详细的流水线以及dockerfile。参考项目源码。

