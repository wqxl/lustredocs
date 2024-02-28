> 系统版本：Centos 8.4.2150  
> 内核版本：4.18.0-305.3.1.el8.x86_64  
> lustre版本：2.15.3  
> backfstype：ldiskfs  

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
## 安装依赖
### 安装内核相关依赖
```bash
yum install kernel-headers-4.18.0-305.3.1.el8.x86_64 kernel-devel-4.18.0-305.3.1.el8 kernel-abi-stablelists-4.18.0-305.3.1.el8 kernel-rpm-macros asciidoc audit binutils clang dwarves java-devel kabi-dw libbabeltrace-devel libbpf-devel libcap-devel libcap-ng-devel libmnl-devel llvm perl-generators python3-docutils
```

### 安装其他编译依赖
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

### 安装e2fsprogs相关依赖
```bash
yum install -y e2fsprogs-1.47.0-wc2 e2fsprogs-devel-1.47.0-wc2 e2fsprogs-libs-1.47.0-wc2 e2fsprogs-static-1.47.0-wc2 libcom_err-1.47.0-wc2 libcom_err-devel-1.47.0-wc2 libss-1.47.0-wc2 libss-devel-1.47.0-wc2
```
e2fsprogs必须要从[https://downloads.whamcloud.com/public/e2fsprogs/1.47.0.wc2/el8/RPMS/x86_64/]()下载，e2fsprogs也是lustre高度定制的。

&nbsp;
&nbsp;
# 编译kernel
## 构建编译环境
### 安装内核源码
```bash
wget https://vault.centos.org/8.4.2105/BaseOS/Source/SPackages/kernel-4.18.0-305.3.1.el8.src.rpm
rpm -ivh kernel-4.18.0-305.3.1.el8.src.rpm
```

### 初始化rpmbuild目录
```bash
cd /root/rpmbuild
rpmbuild -bp --target=`uname -m` ./SPECS/kernel.spec
```

### 修改/root/rpmbuild/BUILD/kernel-4.18.0-305.3.1.el8_4/linux-4.18.0-305.3.1.el8.x86_64/configs/kernel-4.18.0-x86_64.config
**找到带有`# IO Schedulers`的行，在其下面插入以下内容**
```bash
CONFIG_IOSCHED_DEADLINE=y
CONFIG_DEFAULT_IOSCHED="deadline"
```

**将config文件拷贝到/root/rpmbuild/SOURCES**
```bash
cp kernel-4.18.0-x86_64.config /root/rpmbuild/SOURCES
```

**将config文件覆盖lustre kernel config文件**
```bash
cp kernel-4.18.0-x86_64.config /root/rpmbuild/SPECS /root/lustre-2.15.3/lustre/kernel_patches/kernel_configs/kernel-4.18.0-4.18-rhel8.4-x86_64.config
```

### 修改/root/kernel/rpmbuild/SPECS/kernel.spec
**找到带有`find $RPM_BUILD_ROOT/lib/modules/$KernelVer`的行，在其下面插入以下两行内容**
```bash
cp -a fs/ext4/* $RPM_BUILD_ROOT/lib/modules/$KernelVer/build/fs/ext4
rm -f $RPM_BUILD_ROOT/lib/modules/$KernelVer/build/fs/ext4/ext4-inode-test*
```

**找到带有`empty final patch to facilitate testing of kernel patches`的行，在其下面插入以下两行内容**
```bash
# adds Lustre patches
Patch99995: patch-%{version}-lustre.patch
```

**找到带有`ApplyOptionalPatch linux-kernel-test.patch`的行，在其下面插入以下两行内容**
```bash
# lustre patch
ApplyOptionalPatch patch-%{version}-lustre.patch
```

### 打内核补丁
**从lustre源码中生成内核源码补丁**
```bash
cd /root/lustre-2.15.3/lustre/kernel_patches/series
for patch in $(<"4.18-rhel8.series"); do \
    patch_file="/root/lustre-2.15.3/lustre/kernel_patches/patches/${patch}"; \
    cat "${patch_file}" >> "/root/kernel/rpmbuild/SOURCES/lustre-kernel-x86_64-lustre.patch"; \
done
```

&nbsp;
## 编译
### 构建kernel rpms
```bash
rpmbuild -bb --with firmware --target x86_64 kernel.spec
```

### 安装内核并重启
```bash
rpm --reinstall -iUvh kernel-4.18.0-305.3.1.el8.x86_64.rpm kernel-modules-4.18.0-305.3.1.el8.x86_64.rpm kernel-core-4.18.0-305.3.1.el8.x86_64.rpm kernel-devel-4.18.0-305.3.1.el8.x86_64.rpm kernel-headers-4.18.0-305.3.1.el8.x86_64.rpm kernel-modules-extra-4.18.0-305.3.1.el8.x86_64.rpm kernel-tools-4.18.0-305.3.1.el8.x86_64.rpm kernel-tools-libs-4.18.0-305.3.1.el8.x86_64.rpm

reboot
```

&nbsp;
&nbsp;
# 编译lustre
## 构建server rpms
### 配置server
```bash
./configure --enable-server --disable-zfs
```

### 编译
```bash
make rpms -j4
```

&nbsp;
## 构建client rpms
### 清除编译配置
```bash
make distclean
```

### 配置client
```bash
./configure --disable-server --enable-client --disable-zfs
```

### 编译
```bash
make rpms -j4
```