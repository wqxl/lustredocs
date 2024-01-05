> 系统版本：Centos 8.4.2150  
> 内核版本：4.18.0-305.3.1.el8.x86_64  
> lustre版本：2.15.3  
> Pacemaker版本：2.0.5  
> backfstype：zfs  

&nbsp;
本文默认pacemaker和lustre集群已经提前部署。

# 安装软件
### 安装zfs agent
```
cp /opt/ZFS /usr/lib/ocf/resource.d/heartbeat/
```

### 安装ipmi fence
```bash
yum install fence-agents-ipmilan
```

&nbsp;
# 创建resourse
### 创建mgtpool-0 resourse
```bash
pcs resource create mgtpool-0 ocf:heartbeat:ZFS pool="mgtpool-0"
```

### 创建mgt-0 resourse
```bash
pcs resource create mgt-0 ocf:lustre:Lustre \
target="mgtpool-0/mgt-0" \
mountpoint="/lustre/mgt/mgt-0"
```

### 创建mdtpool-0 resourse
```bash
pcs resource create mdtpool-0 ocf:heartbeat:ZFS pool="mdtpool-0"
```

### 创建mdt-0 resourse
```bash
pcs resource create mdt-0 ocf:lustre:Lustre \
target="mdtpool-0/mdt-0" \
mountpoint="/lustre/mdt/mdt-0"
```

### 创建mdtpool-1 resourse
```bash
pcs resource create mdtpool-1 ocf:heartbeat:ZFS pool="mdtpool-1"
```

### 创建mdt-1 resourse
```bash
pcs resource create mdt-1 ocf:lustre:Lustre \
target="mdtpool-0/mdt-1" \
mountpoint="/lustre/mdt/mdt-1"
```

### 创建ostpool-0 resourse
```bash
pcs resource create ostpool-0 ocf:heartbeat:ZFS pool="ostpool-0"
```

### 创建ost-0 resourse
```bash
pcs resource create ost-0 ocf:lustre:Lustre \
target="ostpool-0/ost-0" \
mountpoint="/lustre/ost/ost-0"
```

### 创建ostpool-1 resourse
```bash
pcs resource create ostpool-1 ocf:heartbeat:ZFS pool="ostpool-1"
```

### 创建ost-1 resourse
```bash
pcs resource create ost-1 ocf:lustre:Lustre \
target="ostpool-1/ost-1" \
mountpoint="/lustre/ost/ost-1"
```

&nbsp;
# 配置resourse规则
以下各项规则根据需要配置。

### 添加colocation规则
```bash
pcs constraint colocation add mgt-0 with mgtpool-0 INFINITY
pcs constraint colocation add mdt-0 with mdtpool-0 INFINITY
pcs constraint colocation add mdt-1 with mdtpool-1 INFINITY
pcs constraint colocation add ost-0 with ostpool-0 INFINITY
pcs constraint colocation add ost-1 with ostpool-1 INFINITY
```
上述配置的colocation规则表示：mgt-0 mgtpool-0资源必须在同一个节点上启动，其他类推。如果想要让ostpool-0 mgtpool-0资源绝对不能在同一个节点上启动，只要将`INFINITY`变成`-INFINITY`。

### 添加location规则
```bash
pcs constraint location mgtpool-0 prefers node1=1
pcs constraint location mgtpool-0 prefers node2=2
```
以上配置的location规则表示：mgtpool-0资源优先在node2上启动（pacmaker默认是随机选择节点）。数值越大则优先选择。

### 添加ordering规则
```bash
pcs constraint order set mgtpool-0 mgt-0 sequential=true require-all=true action=start \
                     set ostpool-0 ostpool-1 sequential=false require-all=false action=start \
                     set ost-0 ost-1 sequential=false require-all=false action=start \
                     set mdtpool-0 mdtpool-1 sequential=false require-all=false action=start \
                     set mdt-0 mdt-1 sequential=false require-all=false action=start
```
上述配置的ordering规则表示：mgtpool-0 mgt-0资源是按照顺序启动并且必须全部成功启动，然后启动ostpool-0 ostpool-1资源，启动顺序是无序且无需全部成功启动，然后启动ost-0 ost-1资源，启动顺序是无序且无需全部成功启动，然后启动mdtpool-0 mdtpool-1资源，启动顺序是无序且无需全部成功启动，最后启动mdt-0 mdt-1资源，启动顺序是无序且无需全部成功启动。该规则是将多组ordering set组合成一个ordering，set之间成为了相互依赖关系。如果想要将set独立，只要单独执行`pcs constraint order set`即可。

&nbsp;
# 配置fence
当pacemaker无法停掉某个服务时，可以通过fence强制将该服务所在的机器关机，然后将该服务在其他机器上再次启动。

### 创建fence-ipmi-0 resource
```bash
pcs stonith create fence-ipmi-0 fence_ipmilan \
        ip="192.168.19.64" \
        username="admin" \
        password="admin" \
        pcmk_host_list = "node0"
        pcmk_host_check = "static-list"
```
`fence-ipmi-0`为resource name，`fence_ipmilan`为fence agent名字，fence agent的名字可以通过`pcs stonith list`命令查看，`ip`为ipmi地址，`pcmk_host_list`为ipmi管理的服务器地址。其余参数都是`fence_ipmilan`所支持的options，可以通过`pcs stonith describe fence_ipmilan`命令查看`fence_ipmilan`所有的options。

### 创建fence-ipmi-1 resource
```bash
pcs stonith create fence-ipmi-1 fence_ipmilan \
        ip="192.168.19.64" \
        username="admin" \
        password="admin" \
        pcmk_host_list = "node1"
        pcmk_host_check = "static-list"
```

### 创建fence-ipmi-2 resource
```bash
pcs stonith create fence-ipmi-2 fence_ipmilan \
        ip="192.168.19.64" \
        username="admin" \
        password="admin" \
        pcmk_host_list = "node2"
        pcmk_host_check = "static-list"
```
