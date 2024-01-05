> lustre版本：2.15.3  

&nbsp;
# 简介
nodemap功能支持不同主机上的用户UID和GID映射到MGS服务所在的主机系统上，也就是将所有客户端中所有的用户全部映射到lustre系统中。当不同主机之间存在多个相同UID和GID的用户时，通过映射，可以保证每个用户只能访问自己的文件。

将客户端上的用户映射到MGS服务所在的主机上时，映射后UID和GID在MGS节点上可以存在也可以不存在。如果不存在，在MGS节点上看到的文件属性信息将以映射后的UID和GID呈现。

当开启nodemap功能时，只有加入了nodemap中的客户端并且该客户端中做了UID和GID映射的用户才能访问lustre文件系统，该客户端中的其他未做UID和GID映射的用户是没有权限访问lustre文件系统。对于root这种特权的用户，必须要开启当前nodemap的admin和trusted属性，才有权限访问文件系统。默认情况下admin和trusted属性值是0，也就是root用户没有权限访问lustre文件系统。

nodemap功能相关的设置只能在MGS服务所在的节点上操作。nodemap功能必须通过`lctl nodemap_activate 1`命令开启。如果在nodemap功能生效期间对nodemap做修改，修改不会立即生效，需要等10s。如果想要立即生效，可以重新挂载客户端即可。


&nbsp;
&nbsp;
# 部署
为了所有的操作都能正常执行，lustre文件系统要求提供一个特殊的nodemap，该nodemap必须覆盖所有的服务节点，同时该nodemap必须要开启admin和trusted属性。

## 服务端
### 添加nodemap
```bash
lctl nodemap_add serversmap
```

### 添加NID
```bash
lctl nodemap_add_range --name serversmap --range 192.168.3.[11-13]@tcp
```

### 修改属性
```bash
lctl nodemap_modify --name serversmap --property admin --value 1
lctl nodemap_modify --name serversmap --property trusted --value 1
```

&nbsp;
## 客户端
### 添加nodemap
```bash
lctl nodemap_add clientsmap
```

### 添加NID
```bash
lctl nodemap_add_range --name clientsmap --range 192.168.3.242@tcp
```

### 添加idmap
```bash
lctl nodemap_add_idmap --name clientsmap --idtype uid --idmap 1000:1001
lctl nodemap_add_idmap --name clientsmap --idtype gid --idmap 1000:1001
```

&nbsp;
## 激活nodemap
```bash
lctl nodemap_activate 1
```

&nbsp;
&nbsp;
# 操作
## nodemap相关操作
### 查看所有的nodemap
```bash
lctl nodemap_info
```

### 查看nodemap所有属性
```bash
lctl get_param nodemap.clientsmap.*
```
`clientsmap`是nodemap的名字。

### 删除nodemap
```bash
lctl nodemap_del clientsmap
```
`clientsmap`是nodemap的名字。

### 向nodemap中添加NID
```bash
lctl nodemap_add_range --name clientsmap --range 192.168.3.243@tcp
```
`clientsmap`是nodemap的名字。