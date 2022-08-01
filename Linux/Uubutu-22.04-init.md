## Ubuntu 22.04安装之后的初始化

### 网卡配置

我们想要Ubuntu作为我们的服务器操作系统,这个时候我们需要对其进行静态IP地址的设置.

而且大多数的应用场景下,服务器的IP地址都是固定的.

打开网卡的配置文件进行设置:

```bash
$ sudo vim /etc/netplan/00-installer-config.yaml
```

配置如下:

```yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens32:
      dhcp4: no
      dhcp6: no
      addresses:
        - 192.168.74.103/24
      routes:
        - to: default
          via: 192.168.74.2
      nameservers:
        addresses: [114.114.114.114, 223.5.5.5]
  version: 2
```

说明： addresses \\设置IP地址 

​             routes   \\设置网关和默认路由

​             nameservers  \\设置DNS        

使配置生效

```bash
$ sudo netplan apply
```

### 修改时区配置

时区默认为UTC, 修改为CST; 中国北京时间.

```bash
$ sudo ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

### 配置时间同步

安装chrony,新版的时间同步服务器,与之前的ntpd相同.

```bash
$ sudo apt install chrony -y
```

如果能联网.保持默认配置即可.

如果不能联网的话,需要修改为内部时间服务器. 将`ntp.ubuntu.com` 修改为内部的时间服务器.

```bash
$ sudo systemctl enable chrony
$ sudo /lib/systemd/systemd-sysv-install enable chrony
$ sudo systemctl restart chrony
```



### 配置docker以及kubernetes仓库

说明: 这里之前的`apt-key add -`和`add-apt-repository`添加的软件仓库,将会产生警告.警告内容如下:

```bash
W: https://mirrors.aliyun.com/kubernetes/apt/dists/kubernetes-xenial/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
```

是因为`Ubuntu 22.04` 新添加的源配置文件在`/etc/apt/sources.list.d`中需要单独管理独立的`key`文件位置.

更新`apt`包索引并安装使用kubernetes `apt`仓库所需要的包

```bash
$ sudo apt update
$ sudo apt install -y apt-transport-https ca-certificates curl
```

下载公开签名秘钥,信任Docker的GPG公钥:

```bash
$ sudo curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

添加`docker-ce`软件仓库

```bash
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

下载kuernetes公开签名秘钥,信任kubernetes的GPG公钥

```bash
$ sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg
```

添加 Kubernetes `apt` 仓库

```bash
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```



### 一些基本的优化

这里我们的优化,主要是针对k8s的,有一些优化是通用的.基本上执行,没什么问题.

这里使用root执行.

关闭交换分区

```bash
$ swapoff -a && sysctl -w vm.swappiness=0
$ sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
```

安装依赖工具

```bash
$ apt update && apt install -y wget \
  git \
  psmisc \
  nfs-kernel-server \
  nfs-common \
  jq \
  socat \
  bash-completion \
  ipset \
  ipvsadm \
  conntrack \
  libseccomp2 \
  net-tools \
  cron \
  sysstat \
  unzip \
  dnsutils \
  tcpdump \
  telnet \
  lsof \
  htop \
  curl \
  apt-transport-https \
  ca-certificates
```

这里安装完成之后，会默认启用NFS server，这里我们选择禁用

```bash
$ systemctl stop nfs-server
$ systemctl disable nfs-server
```

配置ipvs开机需要加载的模块，使用`systemd-modules-load`来加载

```bash
$ :> /etc/modules-load.d/ipvs.conf
$ module=(
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
br_netfilter
  )

$ for kernel_module in ${module[@]};do
    /sbin/modinfo -F filename $kernel_module |& grep -qv ERROR && echo $kernel_module >> /etc/modules-load.d/ipvs.conf || :
done
```

使用`systemctl cat systemd-modules-load`看下是否有`Install`段，没有则执行：

```bash
$ cat>>/usr/lib/systemd/system/systemd-modules-load.service<<EOF
[Install]
WantedBy=multi-user.target
EOF
```

启动该模块管理服务

```bash
$ systemctl daemon-reload
$ systemctl enable --now systemd-modules-load.service
$ systemctl restart systemd-modules-load.service
```

确认内核模块加载

```bash
$ lsmod | grep ip_v
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 176128  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          167936  1 ip_vs
nf_defrag_ipv6         24576  2 nf_conntrack,ip_vs
libcrc32c              16384  4 nf_conntrack,btrfs,raid456,ip_vs
```

内核参数优化

```bash
$ cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv4.neigh.default.gc_stale_time = 120
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2
net.ipv4.ip_forward = 1
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2
# 要求iptables不对bridge的数据进行处理
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
net.netfilter.nf_conntrack_max = 2310720
fs.inotify.max_user_watches=89100
fs.may_detach_mounts = 1
fs.file-max = 52706963
fs.nr_open = 52706963
vm.overcommit_memory=1
vm.panic_on_oom=0
vm.swappiness = 0
EOF
```

如果kube-proxy使用ipvs的话，为了防止timeout需要设置下tcp参数。

```bash
$ cat <<EOF >> /etc/sysctl.d/k8s.conf
# https://github.com/moby/moby/issues/31208 
# ipvsadm -l --timout
# 修复ipvs模式下长连接timeout问题 小于900即可
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
EOF
sysctl --system
```

这里修改内核参数，部分会在重启之后失效，比如禁用IPV6.重启之后无法生效。

需要配置相关服务的重新启动。

```bash
$ vim /etc/rc.local
#添加如下内容
#!/bin/bash
# /etc/rc.local

/etc/sysctl.d
/etc/init.d/procps restart

exit 0

#添加执行权限
$ chmod 755 /etc/rc.local
```

优化SSH连接，禁用DNS

```bash
$ sed -ri 's/^#(UseDNS )yes/\1no/' /etc/ssh/sshd_config
```

优化文件最大打开数，在子配置文件中定义

```bash
$ cat>/etc/security/limits.d/kubernetes.conf<<EOF
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

