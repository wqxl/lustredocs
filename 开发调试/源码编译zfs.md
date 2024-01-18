> 系统版本：Centos 8.4.2150  
> 内核版本：4.18.0-305.3.1.el8.x86_64  
> lustre版本：2.15.3  
> backfstype：zfs

&nbsp;
# 准备
## 配置软件仓库源
### 更换centos软件仓库源
```bash
sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' -e 's|^#baseurl=http://mirror.centos.org/$contentdir/$releasever|baseurl=https://vault.centos.org/8.4.2105|g' -i.bak /etc/yum.repos.d/CentOS-*.repo
```

### 添加zfs软件仓库源
```bash
yum install https://zfsonlinux.org/epel/zfs-release-2-3$(rpm --eval "%{dist}").noarch.rpm
```

### 修改zfs软件仓库源
指定centos具体版本号
```bash
baseurl=http://download.zfsonlinux.org/epel/8.4/kmod/$basearch/
```

### 启用必要的软件仓库
```bash
yum config-manager --enable appstream baseos extras powertools ha zfs-kmod
```

### 生成元数据缓存
```bash
yum makecache
```

&nbsp;
## 安装依赖
### 安装内核相关依赖
```bash
yum install -y kernel-devel-4.18.0-305.3.1.el8 kernel-abi-stablelists-4.18.0-305.3.1.el8 kernel-rpm-macros
```
注：kernel-devel和kernel-abi-stablelists必须要和内核版本一致

### 安装编译依赖
```bash
yum install -y bison device-mapper-devel elfutils-devel elfutils-libelf-devel expect \
               flex gcc gcc-c++ git glib2 glib2-devel hmaccalc keyutils-libs-devel \
               krb5-devel ksh libattr-devel libblkid-devel libselinux-devel libtool \
               libuuid-devel libyaml-devel lsscsi make ncurses-devel net-snmp-devel \
               net-tools newt-devel numactl-devel parted patchutils pciutils-devel \
               perl-ExtUtils-Embed pesign redhat-rpm-config rpm-build systemd-devel \
               tcl tcl-devel tk tk-devel wget xmlto yum-utils zlib-devel libmount-devel \
               libnl3-devel python3-devel
```

### 安装zfs
```bash
yum install -y zfs-2.0.5-1.el8 kmod-zfs-2.0.5-1.el8 kmod-zfs-devel-2.0.5-1.el8 libzfs4-devel-2.0.5-1.el8
```
注意：zfs相关的包版本必须要与内核版本适配

&nbsp;
&nbsp;
# 编译
## 源码获取
### 克隆lustre源码
```bash
git clone -b 2.15.3 --depth=1  git://git.whamcloud.com/fs/lustre-release.git lustre-2.15.3
```

### 构建build环境
```bash
bash autogen.sh
```

&nbsp;
## 构建server rpms
### 配置server
```bash
./configure --enable-server --disable-ldiskfs
```

### 编译
```bash
make rpms
```

&nbsp;
## 构建client rpms
### 清除编译配置
```bash
make distclean
```

### 配置client
```bash
./configure --disable-server --enable-client --disable-ldiskfs
```

### 编译
```bash
make rpms
```