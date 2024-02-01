> 系统版本：Centos 8.4.2150  
> 内核版本：4.18.0-305.3.1.el8.x86_64  
> lustre版本：2.15.3  
> zfs版本：2.1.11  

&nbsp;
# 准备
## 配置软件仓库源
### 更换centos软件仓库源
```bash
sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' -e 's|^#baseurl=http://mirror.centos.org/$contentdir/$releasever|baseurl=https://vault.centos.org/8.4.2105|g' -i.bak /etc/yum.repos.d/CentOS-*.repo
```

### 启用必要的软件仓库
```bash
yum config-manager --enable appstream baseos extras powertools ha
```

### 生成元数据缓存
```bash
yum makecache
```

&nbsp;
&nbsp;
# 编译
## zfs编译
lustre 2.15.3发布日志中建议zfs版本为2.1.11或者更高版本，因此需要自己编译zfs。openzfs提供了关于zfs详细文档，详细内容参考[https://openzfs.github.io/openzfs-docs](https://openzfs.github.io/openzfs-docs)。

### 安装zfs编译依赖
```bash
yum install gcc make autoconf automake libtool rpm-build libtirpc-devel libblkid-devel libuuid-devel libudev-devel openssl-devel zlib-devel libaio-devel libattr-devel elfutils-libelf-devel kernel-devel-$(uname -r) yum install python3 python3-devel python3-setuptools python3-cffi libffi-devel git ncompress libcurl-devel python3-packaging
```

### 获取源码
```bash
git clone -b 2.1.11 --depth=1 https://github.com/openzfs/zfs.git zfs-2.1.11
```

### 构建build环境
```bash
bash autogen.sh
```

### 配置
```bash
./configure
```

### 构建 zfs rpms
```bash
make rpms -j4
```

&nbsp;
## lustre编译
lustre官方wiki提供了详细的编译指南，详细内容参考[https://wiki.lustre.org/Compiling_Lustre](https://wiki.lustre.org/Compiling_Lustre)。

### 安装zfs
```bash
yum install sysstat

cd zfs-2.1.11

rpm --reinstall --replacefiles -iUvh zfs-2.1.11-1.el8.x86_64.rpm kmod-zfs-4.18.0-305.3.1.el8.x86_64-2.1.11-1.el8.x86_64.rpm kmod-zfs-devel-4.18.0-305.3.1.el8.x86_64-2.1.11-1.el8.x86_64.rpm kmod-zfs-devel-2.1.11-1.el8.x86_64.rpm libzfs5-2.1.11-1.el8.x86_64.rpm libzfs5-devel-2.1.11-1.el8.x86_64.rpm libnvpair3-2.1.11-1.el8.x86_64.rpm libuutil3-2.1.11-1.el8.x86_64.rpm libzpool5-2.1.11-1.el8.x86_64.rpm
```

### 安装内核相关依赖
```bash
yum install kernel-devel-4.18.0-305.3.1.el8 kernel-abi-stablelists-4.18.0-305.3.1.el8 kernel-rpm-macros
```
注：kernel-devel和kernel-abi-stablelists必须要和内核版本一致

### 安装lustre编译依赖
```bash
yum install bison device-mapper-devel elfutils-devel elfutils-libelf-devel expect flex gcc gcc-c++ git glib2 glib2-devel hmaccalc keyutils-libs-devel krb5-devel ksh libattr-devel libblkid-devel libselinux-devel libtool libuuid-devel libyaml-devel lsscsi make ncurses-devel net-snmp-devel net-tools newt-devel numactl-devel parted patchutils pciutils-devel perl-ExtUtils-Embed pesign redhat-rpm-config rpm-build systemd-devel tcl tcl-devel tk tk-devel wget xmlto yum-utils zlib-devel libmount-devel libnl3-devel python3-devel
```

### 获取源码
```bash
git clone -b 2.15.3 --depth=1  git://git.whamcloud.com/fs/lustre-release.git lustre-2.15.3
```

### 构建build环境
```bash
bash autogen.sh
```

### 配置
```bash
./configure --enable-server --disable-ldiskfs
```
上面是编译server的配置，如果要编译client的配置，修改配置为
```bash
./configure --disable-server --enable-client --disable-ldiskfs
```

### 构建 lustre rpms
```bash
make rpms
```
在执行`make rpms`之前一定要留意，如果前一次和后一次编译不同，比如前一次编译server，后一次编译client，一定要先执行`make distclean`，然后再执行`make rpms`。