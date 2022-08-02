## 基于Ubuntu22.04和Kubeadm部署生产可用的K8s HA集群

### 避免ebtables ethtool 警告

```bash
apt install ebtables ethtool -y
```

### 主机配置

| IP地址         | Hostname   | 最小配置 | Kernel Version                     |
| -------------- | ---------- | -------- | ---------------------------------- |
| 192.168.74.101 | k8s-unode1 | 2CPU 4G  | Linux k8s-unode1 5.15.0-43-generic |
| 192.168.74.102 | k8s-unode2 | 2CPU 4G  | Linux k8s-unode2 5.15.0-43-generic |
| 192.168.74.103 | k8s-unode3 | 2CPU 4G  | Linux k8s-unode3 5.15.0-43-generic |
| 192.168.74.104 | k8s-unode4 | 2CPU 4G  | Linux k8s-unode4 5.15.0-43-generic |

对于Ubuntu安装之前的准备工作,可参考之前的文章,[Ubuntu 22.04最小化安装之后的优化](https://www.toutiao.com/article/7126799367582777856/)



### 安装containerd

下载安装包

```bash
$ cd /usr/local/src/
$ wget https://github.com/containerd/containerd/releases/download/v1.6.6/cri-containerd-cni-1.6.6-linux-amd64.tar.gz
$ tar xzvf cri-containerd-cni-1.6.6-linux-amd64.tar.gz
$ ls -l
total 125608
-rw-r--r-- 1 root root 128619561 Aug  1 16:44 cri-containerd-cni-1.6.6-linux-amd64.tar.gz
drwxr-xr-x 4 root root        51 Jun  7 01:35 etc
drwxr-xr-x 4 root root        35 Jun  7 01:34 opt
drwxr-xr-x 3 root root        19 Jun  7 01:34 usr
```

拷贝

