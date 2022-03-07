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

