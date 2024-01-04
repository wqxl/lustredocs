> 系统版本：Centos 8.4.2150  
> 内核版本：4.18.0-305.3.1.el8.x86_64  
> lustre版本：2.15.3  
> backfstype：ldiskfs  

&nbsp;
# 准备
## 防火墙设置
### 关闭防火墙
```bash
systemctl stop firewalld.service
systemctl disable firewalld.service
```

&nbsp;
## selinux设置
### 关闭selinux
```bash
sed -i.org 's|SELINUX=enforcing|SELINUX=disabled|g' /etc/selinux/config
```

### 重启机器
```bash
reboot
```

### 检查selinux状态
```bash
getenforce
```
如果输出结果是disabled，表明selinux已经关闭。

注：客户端和服务端都需要执行以上所有操作。

&nbsp;
&nbsp;
# 集群部署
## 服务端
### 安装服务端软件
```bash
tar -xzvf lustre-2.15.3-ldiskfs-rpms.tar.gz

cd depends
rpm --reinstall --replacefiles -iUvh *.rpm

cd kernel
rpm --reinstall --replacefiles -iUvh kernel-4.18.0-305.3.1.el8.x86_64.rpm kernel-modules-4.18.0-305.3.1.el8.x86_64.rpm kernel-core-4.18.0-305.3.1.el8.x86_64.rpm kernel-devel-4.18.0-305.3.1.el8.x86_64.rpm kernel-headers-4.18.0-305.3.1.el8.x86_64.rpm kernel-modules-internal-4.18.0-305.3.1.el8.x86_64.rpm kernel-modules-extra-4.18.0-305.3.1.el8.x86_64.rpm

reboot

cd e2fsprogs
rpm --reinstall --replacefiles -iUvh e2fsprogs-1.47.0-wc2.el8.x86_64.rpm e2fsprogs-devel-1.47.0-wc2.el8.x86_64.rpm e2fsprogs-libs-1.47.0-wc2.el8.x86_64.rpm e2fsprogs-static-1.47.0-wc2.el8.x86_64.rpm libcom_err-1.47.0-wc2.el8.x86_64.rpm libcom_err-devel-1.47.0-wc2.el8.x86_64.rpm libss-1.47.0-wc2.el8.x86_64.rpm libss-devel-1.47.0-wc2.el8.x86_64.rpm

cd lustre
rpm --reinstall --replacefiles -iUvh kmod-lustre-2.15.3-1.el8.x86_64.rpm kmod-lustre-osd-ldiskfs-2.15.3-1.el8.x86_64.rpm lustre-2.15.3-1.el8.x86_64.rpm lustre-devel-2.15.3-1.el8.x86_64.rpm lustre-osd-ldiskfs-mount-2.15.3-1.el8.x86_64.rpm lustre-resource-agents-2.15.3-1.el8.x86_64.rpm
```

### 加载lustre内核模块
```
modprobe -v lustre
```

### 配置网络
lustre集群内部通过LNet网络通信，LNet支持InfiniBand and IP networks。本案例采用TCP模式。

**初始化配置lnet**
```bash
lnetctl lnet configure
```
默认情况下`lnetctl lnet configure`会加载第一个up状态的网卡，所以一般情况下不需要再配置net，可以使用`lnetctl net show`命令列出所有的net配置信息，如果没有符合要求的net信息，需要按照下面步骤添加。

**添加tcp**
```bash
lnetctl net add --net tcp --if enp0s8
```
如果`lnetctl lnet configure`已经将添加了tcp，使用`lnetctl net del`删除tcp，然后用`lnetctl net add`重新添加。

**查看添加的tcp**
```bash
lnetctl net show --net tcp
```

**保存到配置文件**
```bash
lnetctl net show --net tcp >> /etc/lnet.conf
```

**开机自启动lnet服务**
```bash
systemctl enable lnet
```

### 部署MGS服务
**创建mgt**
```bash
mkfs.lustre --mgs --backfstype=ldiskfs --reformat /dev/sdb
```

**启动mgs服务**
```bash
mkdir -p /lustre/mgt/mgt-0
mount -t lustre /dev/sdb /lustre/mgt/mgt-0 -v
```
原则上挂载点的名字可以任意取名，建议和mgt名字保持一致。

### 部署MDS服务
**创建mdt**
```bash
mkfs.lustre --mdt --fsname fs00 --index 0 --mgsnode=192.168.3.11@tcp --backfstype=ldiskfs --reformat /dev/sdc
```

**启动mds服务**
```bash
mkdir -p /lustre/mdt/mdt-0
mount -t lustre /dev/sdc /lustre/mdt/mdt-0 -v
```

### 部署OSS服务
**创建ost**
```bash
mkfs.lustre --ost --fsname fs00 --index 0 --mgsnode=192.168.3.11@tcp --backfstype=ldiskfs --reformat /dev/sdd
```

**启动oss服务**
```bash
mkdir -p /lustre/ost/ost-0
mount -t lustre /dev/sdd /lustre/ost/ost-0 -v
```

&nbsp;
## 客户端
### 安装客户端软件
```bash
tar -xzvf lustre-2.15.3-ldiskfs-rpms.tar.gz

cd depends
rpm --reinstall --replacefiles -iUvh perl*.rpm

cd lustre/client
rpm --reinstall --replacefiles -iUvh kmod-lustre-client-2.15.3-1.el8.x86_64.rpm kmod-lustre-client-2.15.3-1.el8.x86_64.rpm
```

### 加载lustre内核模块
```bash
modprobe -v lustre
```

### 配置网络
lustre集群内部通过LNet网络通信，LNet支持InfiniBand and IP networks。本案例采用TCP模式。

**初始化配置lnet**
```bash
lnetctl lnet configure
```
默认情况下`lnetctl lnet configure`会加载第一个up状态的网卡，所以一般情况下不需要再配置net，可以使用`lnetctl net show`命令列出所有的net配置信息，如果没有符合要求的net信息，需要按照下面步骤添加。

**添加tcp**
```bash
lnetctl net add --net tcp --if enp0s8
```
如果`lnetctl lnet configure`已经将添加了tcp，使用`lnetctl net del`删除tcp，然后用`lnetctl net add`重新添加。

**查看添加的tcp**
```bash
lnetctl net show --net tcp
```

**保存到配置文件**
```bash
lnetctl net show --net tcp >> /etc/lnet.conf
```

**开机自启动lnet服务**
```bash
systemctl enable lnet
```

### 挂载文件系统
```bash
mkdir -p /mnt/fs00
mount -t lustre 192.168.3.11@tcp:/fs00 /mnt/fs00 -v
```