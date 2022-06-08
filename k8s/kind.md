## 通过kind安装k8s

kind官网地址：`https://kind.sigs.k8s.io/docs/user/quick-start/`

### 安装docker

由于kind使用的是docker in docker的原理，后续应该会使用containerd。docker本身就会安装containerd。所以这里还是需要安装docker。

基于Ubuntu22.04

```bash
# step 1: 安装GPG证书
sudo curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 2: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 3: 更新并安装Docker-CE
sudo apt -y update
sudo apt -y install docker-ce
```

### 安装kind

这里建议大家安装最新版本即可

```bash
sudo curl -sL https://github.com/kubernetes-sigs/kind/releases/download/v0.14.0/kind-linux-amd64 -o /usr/local/bin/kind
sudo chmod +x /usr/local/bin/kind
sudo kind version
```

 ### 安装kubectl

这里我通过apt源来进行安装，大家也可以自行去GitHub上下载最新版本的kubectl二进制文件。

```
sudo cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
sudo apt update
sudo apt install kubectl
```

### 准备就绪，创建集群

```bash
#这里默认创建的集群名称为kind
sudo kind create cluster --wait 10m 
#也可以指定集群名称
sudo kind create cluster --name my-cluster
#由于这里我后安装了kubectl，导致没有kubeconfig配置，使用如下命令导出kubeconfig
sudo kind export kubeconfig
------------------------------
iqimei@iqimei-vm:~/src$ sudo kubectl get pod -n kube-system
NAME                                         READY   STATUS    RESTARTS   AGE
coredns-6d4b75cb6d-5mhwd                     1/1     Running   0          12m
coredns-6d4b75cb6d-5xwb4                     1/1     Running   0          12m
etcd-kind-control-plane                      1/1     Running   0          13m
kindnet-55pxh                                1/1     Running   0          12m
kube-apiserver-kind-control-plane            1/1     Running   0          13m
kube-controller-manager-kind-control-plane   1/1     Running   0          13m
kube-proxy-bt4xt                             1/1     Running   0          12m
kube-scheduler-kind-control-plane            1/1     Running   0          13m

iqimei@iqimei-vm:~/src$ sudo kubectl get node
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   53m   v1.24.0
```



### 删除创建的集群，创建多节点集群

```bash
#删除集群
sudo kind delete cluster --name kind
#多节点集群的配置文件如下：
iqimei@iqimei-vm:~/src$ cat kind-3nodes.yaml 
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: app-1-cluster
nodes:
  - role: control-plane
    extraMounts:
    - hostPath: /etc/hosts
      containerPath: /etc/hosts
      readOnly: true
      selinuxRelabel: false
      propagation: HostToContainer
  - role: worker
    extraMounts:
    - hostPath: /etc/hosts
      containerPath: /etc/hosts
      readOnly: true
      selinuxRelabel: false
      propagation: HostToContainer
  - role: worker
    extraMounts:
    - hostPath: /etc/hosts
      containerPath: /etc/hosts
      readOnly: true
      selinuxRelabel: false
      propagation: HostToContainer
 ---
 
#创建集群
kind create cluster --config kind-example-config.yaml
#获取集群信息
sudo kubectl cluster-info --context kind-app-1-cluster
#集群配置kubeconfig默认存放再/root/.kube/config当中
```

### Kind 的镜像

1. BASE镜像

   官方会基于Ubuntu新版本ubuntu:21.10；作为基础镜像，并进行如下主要调整:

   - 安装 `Systemd` 相关的包，并调整一些配置以适应在容器内运行

   - 安装 `Kubernetes` 运行时的依赖包，比如: `Conntrack`、`Socat`、`CNI` 等

   - 安装容器运行环境，比如: `Containerd`、`Crictl` 等

   - 配置自己的 `ENTRYPOINT` 脚本，以适应和调整容器内运行的问题

     更多具体的构建逻辑可以参考：https://github.com/kubernetes-sigs/kind/blob/master/images/base/Dockerfile

2. Node镜像

   `Node` 镜像的构建比较复杂，目前是通过运行 `Base` 镜像并在 `Base` 镜像内执行操作，再保存此容器内容为镜像的方式来构建的，包含的操作有：

   - 构建 `Kubernetes` 相关资源，比如：二进制文件和镜像
   - 运行一个用于构建的容器
   - 把构建的 `Kubernetes` 相关资源复制到容器里
   - 调整部分组件配置参数，以支持在容器内运行
   - 预先拉去运行环境需要的镜像
   - 通过 `docker commit` 方式保存当前的构建容器为 `Node` 镜像