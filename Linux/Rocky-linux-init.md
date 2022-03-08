## Rocky Linux安装之后初始化

### 网络

#### 修改IP

```
vi /etc/sysconfig/network-scripts/ifcfg-ens32
TYPE=Ethernet
BOOTPROTO=none
DEVICE=ens32
ONBOOT=yes
IPADDR=192.168.10.87
NETMASK=255.255.255.0
GATEWAY=192.168.10.1
DNS1=114.114.114.114
```

修改配置文件后，使配置生效,可使用tab键进行补全。

```bash
nmcli connection reload  #重载所有ifcfg或route到connection
nmcli device reapply ens32 #使其生效
nmcli device connect ens32 #功能同上
nmcli device #查看device列表
nmcli device show #查看所有详细信息
```

### 防火墙

#### 关闭selinux

```
setenforce 0
sed -ri '/^[^#]*SELINUX=/s#=.+$#=disabled#' /etc/selinux/config
```

#### 关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
```



### 时间

#### 时区修改

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

#### 配置时间同步

```bash
dnf install chrony -y #安装chrony，可能默认已安装
#修改配置文件
vi /etc/chrony.conf
#修改pool 2.pool.ntp.org iburst 
pool ntp.aliyun.com iburst
#然后进行启动
systemctl enable chronyd 
systemctl start chronyd
systemctl status chronyd
```

### 镜像源

#### 基础镜像源

```bash
sed -e 's|^mirrorlist=|#mirrorlist=|g' \
    -e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.aliyun.com/rockylinux|g' \
    -i.bak \
    /etc/yum.repos.d/Rocky-*.repo
```

#### epel源

```bash
dnf install epel-release -y
sed -i 's|^#baseurl=https://download.example/pub|baseurl=https://mirrors.aliyun.com|' /etc/yum.repos.d/epel*
sed -i 's|^metalink|#metalink|' /etc/yum.repos.d/
```

### ulimt参数优化

修改文件句柄数优化,需要重启。

```bash
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



### k8s基础配置

#### 内核参数优化

```bash
#关闭交换分区
swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
```

```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
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
# https://github.com/moby/moby/issues/31208 
# ipvsadm -l --timout
# 修复ipvs模式下长连接timeout问题 小于900即可
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
EOF
sysctl --system
```

#### 添加ipvs模块支持

安装依赖包

```bash
dnf install -y \
    curl \
    wget \
    git \
    parted \
    conntrack-tools \
    psmisc \
    nfs-utils \
    jq \
    socat \
    ebtables \
    ethtool \
    util-linux \
    bash-completion \
    ipset \
    ipvsadm \
    conntrack \
    libseccomp \
    net-tools \
    crontabs \
    sysstat \
    unzip \
    netcat \
    iftop \
    nload \
    strace \
    bind-utils \
    tcpdump \
    telnet \
    lsof \
    htop
```

如果集群kube-proxy想使用ipvs模式的话需要开机加载下列模块儿，按照规范使用`systemd-modules-load`来加载而不是在`/etc/rc.local`里写modprobe

```bash
:> /etc/modules-load.d/ipvs.conf
module=(
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
br_netfilter
  )
for kernel_module in ${module[@]};do
    /sbin/modinfo -F filename $kernel_module |& grep -qv ERROR && echo $kernel_module >> /etc/modules-load.d/ipvs.conf || :
done
```

先使用`systemctl cat systemd-modules-load`看下有没有`Install`段，没有则执行下面:

```bash
cat>>/usr/lib/systemd/system/systemd-modules-load.service<<EOF
[Install]
WantedBy=multi-user.target
EOF
```

启动该模块管理服务

```bash
systemctl daemon-reload
systemctl enable systemd-modules-load
systemctl start systemd-modules-load
```

确认内核模块加载

```bash
lsmod | grep ip_v
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 172032  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          172032  1 ip_vs
nf_defrag_ipv6         20480  2 nf_conntrack,ip_vs
libcrc32c              16384  3 nf_conntrack,xfs,ip_vs
```

