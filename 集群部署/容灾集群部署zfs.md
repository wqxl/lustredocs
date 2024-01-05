> 系统版本：Centos 8.4.2150  
> 内核版本：4.18.0-305.3.1.el8.x86_64  
> lustre版本：2.15.3  
> backfstype：zfs  

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
yum install lustre-osd-zfs-mount lustre lustre-resource-agents zfs
```

### 加载zfs和lustre内核模块
```
modprobe -v zfs
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
**创建mgtpool**
```bash
zpool create -f -O canmount=off -o multihost=on -o cachefile=none mgtpool-0 /dev/sdb
```
容灾模式下使用zpool创建pool时，必须要开启了multihost功能支持。multihost要求为每一个host提供不同的hostid，如果不提供，该命令执行失败。在每一个host上执行`zgenhostid $(hostid)`便可以生成不同的hostid。  
使用`zpool`创建pool池可以同时绑定多个磁盘，并采用raid0模式来存储数据。如果需要对pool扩容，必须使用`zpool add`添加磁盘到指定的pool中。

**创建mgt**
```bash
mkfs.lustre --mgs \
--servicenode=192.168.3.11@tcp \
--servicenode=192.168.3.12@tcp \
--backfstype=zfs \
--reformat mgtpool-0/mgt-0
```
`mgtpool-0/mgt-0`是`mgtpool-0`的一个逻辑卷，逻辑卷的数量和容量都是可以通过`zfs`命令控制。  
`servicenode`参数指定当前创建的mgt能够在哪些节点上被使用(容灾)。该参数的数量没有限制。可以将多个`servicenode`参数合并成一个，比如上面的参数可以改写成`--servicenode=192.168.3.11@tcp:192.168.3.12@tcp`。

**启动mgs服务**
```bash
mkdir -p /lustre/mgt/mgt-0
mount -t lustre mgtpool-0/mgt-0 /lustre/mgt/mgt-0 -v
```

### 部署MDS服务
#### Primary Node1
**创建mdtpool**
```bash
zpool create -f -O canmount=off -o multihost=on -o cachefile=none mdtpool-0 /dev/sdc
```

**创建mdt**
```bash
mkfs.lustre --mdt \
--fsname fs00 \
--index 0 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.11@tcp \
--servicenode 192.168.3.12@tcp \
--backfstype=zfs \
--reformat mdtpool-0/mdt-0
```
如果mgs服务有多个，必须要同时指定多个mgsnode，而且第一个mgsnode必须是primary mgs。另外对于每一个lustre文件系统，mdt index序号必须从0开始，0代表整个文件系统的根目录。  
`mdtpool-0/mdt-0`是`mdtpool-0`的一个逻辑卷，使用`mount`挂载一个逻辑卷，表示启动一个mds服务。如果想要在同一个节点上启动多个mds，则需要在`mdtpool-0`中再申请一个逻辑卷，此时`--reformat`参数可以省略，`--index`必须递增。一个mds可以同时管理多个逻辑卷，只需要在`--reformat`参数后同时指定多个逻辑卷。

**启动mds服务**
```bash
mkdir -p /lustre/mdt/mdt-0
mount -t lustre mdtpool-0/mdt-0 /lustre/mdt/mdt-0 -v
```

#### Primary Node2
**创建mdspool**
```bash
zpool create -f -O canmount=off -o multihost=on -o cachefile=none mdtpool-1 /dev/sdb
```

**创建mdt**
```bash
mkfs.lustre --mdt \
--fsname fs00 \
--index 1 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.12@tcp \
--servicenode 192.168.3.11@tcp \
--backfstype=zfs \
--reformat mdtpool-1/mdt-1
```
如果mgs服务有多个，必须要同时指定多个mgsnode，而且第一个mgsnode必须是primary mgs。另外对于每一个lustre文件系统，mdt index序号必须从0开始，0代表整个文件系统的根目录。

**启动mds服务**
```bash
mkdir -p /lustre/mdt/mdt-1
mount -t lustre mdtpool-1/mdt-1 /lustre/mdt/mdt-1 -v
```

### 部署OSS服务
#### Primary Node1
**创建ostpool**
```bash
zpool create -f -O canmount=off -o multihost=on -o cachefile=none ostpool-0 /dev/sdd
```

**创建ost**
```bash
mkfs.lustre --ost \
--fsname fs00 \
--index 0 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.11@tcp \
--servicenode 192.168.3.12@tcp \
--backfstype=zfs \
--reformat ostpool-0/ost-0
```

**启动oss服务**
```bash
mkdir -p /lustre/ost/ost-0
mount -t lustre ostpool-0/ost-0 /lustre/ost/ost-0 -v
```

#### Primary Node2
**创建ostpool**
```bash
zpool create -f -O canmount=off -o multihost=on -o cachefile=none ostpool-1 /dev/sdc
```

**创建ost**
```bash
mkfs.lustre --ost \
--fsname fs00 \
--index 1 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.12@tcp \
--servicenode 192.168.3.11@tcp \
--backfstype=zfs \
--reformat ostpool-1/ost-1
```

**启动oss服务**
```bash
mkdir -p /lustre/ost/ost-1
mount -t lustre ostpool-1/ost-1 /lustre/ost/ost-1 -v
```

&nbsp;
## 客户端
### 安装客户端软件
```bash
yum install lustre-client
```

### 加载lustre内核模块
```bash
modprobe lustre
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