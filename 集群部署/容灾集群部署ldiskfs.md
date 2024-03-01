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
yum reinstall kernel kernel-modules kernel-core kernel-tools kernel-tools-libs
yum install e2fsprogs e2fsprogs-libs
yum install kmod-lustre kmod-lustre-osd-ldiskfs lustre lustre-osd-ldiskfs-mount lustre-iokit lustre-resource-agents
```
也可以使用`rpm --reinstall --replacefiles -iUvh xxxx.rpm`命令离线安装。

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
注：所有的服务端都需要执行以上操作。

&nbsp;
### 部署MGS服务
#### Primary Node1
**创建mgt**
```bash
mkfs.lustre --mgs \
--servicenode=192.168.3.11@tcp \
--servicenode=192.168.3.12@tcp \
--backfstype=ldiskfs \
--reformat /dev/sdb
```
`servicenode`参数指定当前创建的mgt能够在哪些节点上被使用(容灾)。该参数的数量没有限制。可以将多个`servicenode`参数合并成一个，比如上面的参数可以改写成`--servicenode=192.168.3.11@tcp:192.168.3.12@tcp`。

**启动mgs服务**
```bash
mkdir -p /lustre/mgt/mgt-0000
mount -t lustre /dev/disk/by-label/MGS /lustre/mgt/mgt-0000 -v
```

### 部署MDS服务
#### Primary Node1
**创建mdt**
```bash
mkfs.lustre --mdt \
--fsname fs00 \
--index 0x00 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.11@tcp \
--servicenode 192.168.3.12@tcp \
--backfstype=ldiskfs \
--reformat /dev/sdc
```
如果mgs服务有多个，必须要同时指定多个mgsnode，而且第一个mgsnode必须是primary mgs。另外对于每一个lustre文件系统，mdt index序号必须从0开始，0代表整个文件系统的根目录。

**启动mds服务**
```bash
mkdir -p /lustre/mdt/mdt-0000
mount -t lustre /dev/disk/by-label/fs00-MDT0000 /lustre/mdt/mdt-0000 -v
```

#### Primary Node2
**创建mdt**
```bash
mkfs.lustre --mdt \
--fsname fs00 \
--index 0x01 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.12@tcp \
--servicenode 192.168.3.11@tcp \
--backfstype=ldiskfs \
--reformat /dev/sdd
```
如果mgs服务有多个，必须要同时指定多个mgsnode，而且第一个mgsnode必须是primary mgs。另外对于每一个lustre文件系统，mdt index序号必须从0开始，0代表整个文件系统的根目录。

**启动mds服务**
```bash
mkdir -p /lustre/mdt/mdt-0001
mount -t lustre /dev/disk/by-label/fs00-MDT0001 /lustre/mdt/mdt-0001 -v
```

### 部署OSS服务
#### Primary Node1
**创建ost**
```bash
mkfs.lustre --ost \
--fsname fs00 \
--index 0x00 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.11@tcp \
--servicenode 192.168.3.12@tcp \
--backfstype=ldiskfs \
--reformat /dev/sde
```

**启动oss服务**
```bash
mkdir -p /lustre/ost/ost-0000
mount -t lustre /dev/disk/by-label/fs00-OST0000 /lustre/ost/ost-0000 -v
```

#### Primary Node2
**创建ost**
```bash
mkfs.lustre --ost \
--fsname fs00 \
--index 0x01 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.12@tcp \
--servicenode 192.168.3.11@tcp \
--backfstype=ldiskfs \
--reformat /dev/sdf
```

**启动oss服务**
```bash
mkdir -p /lustre/ost/ost-0001
mount -t lustre /dev/disk/by-label/fs00-OST0001 /lustre/ost/ost-0001 -v
```

&nbsp;
## 客户端
### 安装客户端软件
```bash
yum install kmod-lustre-client lustre-client lustre-iokit
```
也可以使用`rpm --reinstall --replacefiles -iUvh xxxx.rpm`命令离线安装。

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
注：所有的客户端都需要执行以上操作。

### 挂载文件系统
```bash
mkdir -p /mnt/fs00
mount -t lustre 192.168.3.11@tcp:192.168.3.12@tcp:/fs00 /mnt/fs00 -v
```