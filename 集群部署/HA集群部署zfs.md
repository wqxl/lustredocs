> 系统版本：Centos 8.4.2150  
> 内核版本：4.18.0-305.3.1.el8.x86_64  
> lustre版本：2.15.3  
> Pacemaker版本：2.0.5  
> backfstype：zfs  

&nbsp;
本文默认pacemaker和lustre集群已经提前部署。

# 集群规划
```bash
mgt-0000     192.168.3.11     node1
mdt-0000     192.168.3.11     node1
mdt-0001     192.168.3.12     node2
ost-0000     192.168.3.13     node3
ost-0001     192.168.3.14     node4
```
node1与node2互为主备，node3与node4互为主备。

&nbsp;
&nbsp;
# 安装软件
### 安装zfs agent
```
cp /opt/ZFS /usr/lib/ocf/resource.d/heartbeat/
```

### 查看zfs agent是否已经安装
```bash
pcs resource agents ocf:heartbeat
```
如果输出中有ZFS，说明已经安装。或者执行`pcs resource list ocf:heartbeat:ZFS`也可以查看。

&nbsp;
# 创建resourse
### 创建mgtpool-0000资源
```bash
pcs resource create mgtpool-0000 ocf:heartbeat:ZFS pool="mgtpool-0000"
```

### 创建mgt-0000资源
```bash
pcs resource create mgt-0000 ocf:lustre:Lustre \
target="mgtpool-0000/mgt-0000" \
mountpoint="/lustre/mgt/mgt-0000"
```

### 创建mdtpool-0000资源
```bash
pcs resource create mdtpool-0000 ocf:heartbeat:ZFS pool="mdtpool-0000"
```

### 创建mdt-0000资源
```bash
pcs resource create mdt-0000 ocf:lustre:Lustre \
target="mdtpool-0000/mdt-0000" \
mountpoint="/lustre/mdt/mdt-0000"
```

### 创建mdtpool-0001资源
```bash
pcs resource create mdtpool-0001 ocf:heartbeat:ZFS pool="mdtpool-0001"
```

### 创建mdt-0001资源
```bash
pcs resource create mdt-0001 ocf:lustre:Lustre \
target="mdtpool-0001/mdt-0001" \
mountpoint="/lustre/mdt/mdt-0001"
```

### 创建ostpool-0000资源
```bash
pcs resource create ostpool-0000 ocf:heartbeat:ZFS pool="ostpool-0000"
```

### 创建ost-0000资源
```bash
pcs resource create ost-0000 ocf:lustre:Lustre \
target="ostpool-0000/ost-0000" \
mountpoint="/lustre/ost/ost-0000"
```

### 创建ostpool-0001资源
```bash
pcs resource create ostpool-0001 ocf:heartbeat:ZFS pool="ostpool-0001"
```

### 创建ost-0001资源
```bash
pcs resource create ost-0001 ocf:lustre:Lustre \
target="ostpool-0001/ost-0001" \
mountpoint="/lustre/ost/ost-0001"
```

&nbsp;
# 配置resourse规则
以下各项规则根据需要配置。

### 添加location规则
```bash
pcs constraint location mgtpool-0000 prefers node1=INFINITY node2=INFINITY node3=-INFINITY node4=-INFINITY
pcs constraint location mdtpool-0000 prefers node1=INFINITY node2=INFINITY node3=-INFINITY node4=-INFINITY
pcs constraint location mdtpool-0001 prefers node1=INFINITY node2=INFINITY node3=-INFINITY node4=-INFINITY
pcs constraint location ostpool-0000 prefers node1=-INFINITY node2=-INFINITY node3=INFINITY node4=INFINITY
pcs constraint location ostpool-0001 prefers node1=-INFINITY node2=-INFINITY node3=INFINITY node4=INFINITY
```
以上配置的location规则表示：mgt-0000资源在node1和node2之间随机选择（pacmaker默认是随机选择节点），并且绝不选择node3和node4。数值相等时是随机选择，数值不相等时，优先选择数值大的节点。

### 添加colocation规则
```bash
pcs constraint colocation add mgt-0000 with mgtpool-0000 INFINITY
pcs constraint colocation add mdt-0000 with mdtpool-0000 INFINITY
pcs constraint colocation add mdt-0001 with mdtpool-0001 INFINITY
pcs constraint colocation add ost-0000 with ostpool-0000 INFINITY
pcs constraint colocation add ost-0001 with ostpool-0001 INFINITY
```
上述配置的colocation规则表示：mgt-0000 mgtpool-0000资源必须在同一个节点上启动，其他类推。如果想要让mgt-0000 mgtpool-0000资源绝对不能在同一个节点上启动，只要将`INFINITY`变成`-INFINITY`。

### 添加ordering规则
```bash
pcs constraint order set mgtpool-0000 mgt-0000 sequential=true require-all=true action=start \
                     set ostpool-0000 ostpool-0001 sequential=false require-all=false action=start \
                     set ost-0000 ost-0001 sequential=false require-all=false action=start \
                     set mdtpool-0000 mdtpool-0001 sequential=false require-all=false action=start \
                     set mdt-0000 mdt-0001 sequential=false require-all=false action=start
```
上述配置的ordering规则表示：mgtpool-0000 mgt-0000资源是按照顺序启动并且必须全部成功启动，然后启动ostpool-0000 ostpool-0001资源，启动顺序是无序且无需全部成功启动，然后启动ost-0000 ost-0001资源，启动顺序是无序且无需全部成功启动，然后启动mdtpool-0000 mdtpool-0001资源，启动顺序是无序且无需全部成功启动，最后启动mdt-0000 mdt-0001资源，启动顺序是无序且无需全部成功启动。该规则是将多组ordering set组合成一个ordering，set之间成为了相互依赖关系。如果想要将set独立，只要单独执行`pcs constraint order set`即可。

&nbsp;
# 配置fence
当pacemaker无法停掉某个服务时，可以通过fence强制将该服务所在的机器关机，然后将该服务在其他机器上再次启动。
### 开启stonith-enabled
```bash
pcs property set stonith-enabled=true
```
### 创建fence-ipmi-0000 resource
```bash
pcs stonith create fence-ipmi-0000 fence_ipmilan \
        ip="192.168.19.64" \
        username="admin" \
        password="admin" \
        pcmk_host_list = "node1"
        pcmk_host_check = "static-list"
```
`fence-ipmi-0`为resource name，`fence_ipmilan`为fence agent名字，fence agent的名字可以通过`pcs stonith list`命令查看，`ip`为ipmi地址，`pcmk_host_list`为ipmi管理的服务器地址。其余参数都是`fence_ipmilan`所支持的options，可以通过`pcs stonith describe fence_ipmilan`命令查看`fence_ipmilan`所有的options。

### 创建fence-ipmi-0001 resource
```bash
pcs stonith create fence-ipmi-0001 fence_ipmilan \
        ip="192.168.19.64" \
        username="admin" \
        password="admin" \
        pcmk_host_list = "node2"
        pcmk_host_check = "static-list"
```

### 创建fence-ipmi-0002 resource
```bash
pcs stonith create fence-ipmi-0002 fence_ipmilan \
        ip="192.168.19.64" \
        username="admin" \
        password="admin" \
        pcmk_host_list = "node3"
        pcmk_host_check = "static-list"
```

### 创建fence-ipmi-0003 resource
```bash
pcs stonith create fence-ipmi-0002 fence_ipmilan \
        ip="192.168.19.65" \
        username="admin" \
        password="admin" \
        pcmk_host_list = "node4"
        pcmk_host_check = "static-list"
```