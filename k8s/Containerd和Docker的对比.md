## containerd和Docker作为k8s runtime的对比



### 运行时部署的结构对比

[参考链接](https://help.aliyun.com/document_detail/160313.html)

| 运行时     | 调用结构                                                     |
| ---------- | ------------------------------------------------------------ |
| Docker     | kubelet -> dockerd -> containerd -> containerd-shim -> runC容器 |
| Containerd | kubelet -> containerd -> containerd-shim -> runC容器         |

## Docker和Containerd两种容器引擎常用命令对比

Docker运行时和安全沙箱运行时的容器引擎分别是Docker和Containerd。这两种容器引擎都有各自的命令工具来管理镜像和容器。两种容器引擎常用命令对比如下。

| 命令                 | Docker           | Containerd        |                          |
| :------------------- | :--------------- | :---------------- | ------------------------ |
|                      | docker           | crictl（推荐）    | ctr                      |
| 查看容器列表         | `docker ps`      | `crictl ps`       | `ctr -n k8s.io c ls`     |
| 查看容器详情         | `docker inspect` | `crictl inspect`  | `ctr -n k8s.io c info`   |
| 查看容器日志         | `docker logs`    | `crictl logs`     | 无                       |
| 容器内执行命令       | `docker exec`    | `crictl exec`     | 无                       |
| 挂载容器             | `docker attach`  | `crictl attach`   | 无                       |
| 显示容器资源使用情况 | `docker stats`   | `crictl stats`    | 无                       |
| 创建容器             | `docker create`  | `crictl create`   | `ctr -n k8s.io c create` |
| 启动容器             | `docker start`   | `crictl start`    | `ctr -n k8s.io run`      |
| 停止容器             | `docker stop`    | `crictl stop`     | 无                       |
| 删除容器             | `docker rm`      | `crictl rm`       | `ctr -n k8s.io c del`    |
| 查看镜像列表         | `docker images`  | `crictl images`   | `ctr -n k8s.io i ls`     |
| 查看镜像详情         | `docker inspect` | `crictl inspecti` | 无                       |
| 拉取镜像             | `docker pull`    | `crictl pull`     | `ctr -n k8s.io i pull`   |
| 推送镜像             | `docker push`    | 无                | `ctr -n k8s.io i push`   |
| 删除镜像             | `docker rmi`     | `crictl rmi`      | `ctr -n k8s.io i rm`     |
| 查看Pod列表          | 无               | `crictl pods`     | 无                       |
| 查看Pod详情          | 无               | `crictl inspectp` | 无                       |
| 启动Pod              | 无               | `crictl runp`     | 无                       |
| 停止Pod              | 无               | `crictl stopp`    | 无                       |

### 容器日志以及相关参数



[参考链接](https://cloud.tencent.com/developer/article/1450788)

| 对比项目               | Docker                                                       | Containerd                                                   |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 存储路径               | docker作为k8s容器运行时的情况下，容器日志的落盘由docker来完成， 保存在类似`/var/lib/docker/containers/$CONTAINERID`目录下。kubelet会在`/var/log/pods`和`/var/log/containers`下面建立软链接，指向`/var/lib/docker/containers/$CONTAINERID`目录下的容器日志文件 | containerd作为k8s容器运行时的情况下， 容器日志的落盘由kubelet来完成，保存到`/var/log/pods/$CONTAINER_NAME`目录下，同时在`/var/log/containers`目录下创建软链接，指向日志文件 |
| 配置参数               | 在docker配置文件中指定：    "log-driver": "json-file",     "log-opts": {"max-size": "100m","max-file": "5"} | 方法一：在kubelet参数中指定：  --container-log-max-files=5 --container-log-max-size="100Mi"  方法二：在KubeletConfiguration中指定：    "containerLogMaxSize": "100Mi",    "containerLogMaxFiles": 5, |
| 把容器日志保存到数据盘 | 把数据盘挂载到"data-root"(缺省是/var/lib/docker)即可         | 创建一个软链接/var/log/pods指向数据盘挂载点下的某个目录  在TKE中选择"将容器和镜像存储在数据盘"，会自动创建软链接/var/log/pods |

### CNI比对

| 对比项        | docker                                      | containerd                                                   |
| ------------- | ------------------------------------------- | ------------------------------------------------------------ |
| 谁负责调用CNI | kubelet内部的docker-shim                    | containerd内置的cri-plugin(containerd 1.1以后)               |
| 如何配置CNI   | kubelet参数 --cni-bin-dir 和 --cni-conf-dir | containerd配置文件(toml)： plugins.cri.cni    bin_dir = "/opt/cni/bin"    conf_dir = "/etc/cni/net.d" |

