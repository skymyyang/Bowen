### Redhat系列配置本地YUM源

1. 确定下载的ISO镜像为DVD版本，包含软件包

2. 配置yum源

   ```bash
   [root@localhost ~]# umount /dev/cdrom
   [root@localhost ~]# mount /dev/cdrom /mnt/
   [root@localhost ~]# cat /etc/yum.repos.d/rhel7.repo 
   [rhel7]
   name=rhel7
   baseurl=file:///mnt
   enabled=1
   gpgcheck=0
   [root@localhost ~]# 
   ```
   
   

