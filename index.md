# 一些下载链接
[Fedora-Server-DVD-x86_64-23.iso](http://10.128.201.9:8181/owncloud/index.php/s/fT8tN4UabpXt6Ec)

[VM及教程](http://10.128.201.9:8181/owncloud/index.php/s/piViJ1MbGz1habM)

# GlusterFS部署实现

## 一、简要介绍

Gluster是一个分布式文件系统，具有很高的横向扩展能力。它的实现并不基于一个集中化的元数据服务器，因此没有单点故障等问题。在GlusterFS中存储的数据，可通过传统的NFS，SMB/CIFS、FUSE等协议，在Windows、Linux系统中访问GlusterFS集群。

## 二、实验目标与要求

1）在Vmware9中，使用Fedora-Server-DVD-x86_64-23.iso安装2至4个Fedora虚拟机。
2）分别在每个节点上部署Fedora。
3）在Linux系统上实现Native挂载；在Windows7 企业版以上实现NFS挂载。

## 三、部署过程

以下共由2个节点构成GLusterFS集群。一个节点的IP地址为172.16.77.135。另一个是172.16.77.136。具体需要根据各自的实验环境而确定。

### 1.在Vmware中，为每个虚拟机增加一块磁盘。
通过命令`cat /proc/partitions`查看，此时新增了一块磁盘`/dec/sdb`。
![虚拟机增加磁盘](http://7xilc8.com1.z0.glb.clouddn.com/tmp1.png)

### 2.固定虚拟机IP地址。
A）使用`ifconfig`命令，查看当前网卡名称，如`ifcfg-ens33`。

B）使用如下命令，修改`ifcfg-ens33`，以固定IP：
```
cd /etc/sysconfig/network-scripts
ls
vi ifcfg-ens33
```
在适当位置添加：
```
IPADDR=172.16.77.135
NETMASK=255.255.255.0
GATEWAY=172.16.77.2
DNS1=172.16.77.2
```
修改BOOTPROTO项为`static`：
```
BOOTPROTO="static"
```
确保ONBOOT项为`yes`：
```
ONBOOT="yes"
```

C）如果采用克隆方式生成的虚拟机，需要特别进行如下操作：
1）使用`ifconfig`查看网卡的MAC地址
2）在`/etc/sysconfig/network-scripts`文件中，修改MAC地址栏内容（HWADDR行）。

D）通过以下命令重启网络，或重启虚拟机（3选1）：
```
service network restart

service network stop
service network start

init 0				#关机
init 6				#重启
shutdown -h now	#关机
```

### 3.格式化、挂载分区（bricks）
假设分区位于`/dev/sdb`。
在每个节点上执行以下命令：
```
mkfs.xfs -i size=512 /dev/sdb
mkdir -p /data/brick1
echo '/dev/sdb /data/brick1 xfs defaults 1 2' >> /etc/fstab
mount -a && mount
```

### 4.安装GlusterFS
在每个节点上执行以下命令，以安装GlusterFS：
```
yum install glusterfs-server
```
启动GlusterFS管理守护进程：
```
service glusterd start
service glusterd status
```

### 5.配置可信任池
A）首先，使用`iptables -F`命令，关闭每个虚拟机的防火墙。

B）在其中一个节点上，使用如下命令，探测其他所有机器：
```
gluster peer probe 172.16.77.136
gluster peer probe 172.16.77.137
gluster peer probe 172.16.77.138
```
具体根据所要搭建的节点数量，而不同。

### 6.建立GlusterFS卷
```
mkdir /data/brick1/gv0
```
在任一个节点上运行：
```
gluster volume create gv0 replica 2 172.16.77.135:/data/brick1/gv0 172.16.77.136:/data/brick1/gv0
gluster volume start gv0
```
确认卷的信息：
```
gluster volume info
```

在每台机器上，运行以下命令：
```
service glusterd stop
service rpcbind start	#开启rpcbind服务
service glusterd start	#必须重启glusterd服务
```
或
```
service rpcbind start
service glusterd restart
```

### 7.测试GlusterFS卷
将使用其中一个服务节点，挂载该卷。通常，我们从外部的机器中挂载该卷，成为客户端。由于该方法需要安装额外的包到客户端机器上，因此我们将使用其中一个服务节点作为“客户端”，进行简单的测试。

在服务节点1上，挂载创建出来的gv0卷：
```
mount -t glusterfs 172.16.77.135:gv0 /mnt
```
创建100个文件：
```
for i in `seq -w 1 100`; do cp -rp /var/log/messages /mnt/copy-test-$i; done
```

检查挂载点，此时将会看到返回了100个文件（还有1个.glusterfs目录）：
```
ls -lA /mnt | wc -l
``` 

检查在每个服务节点上的GlusterFS挂载点：
```
ls -lA /data/brick1/gv0
```
在2个服务节点上执行上述语句时，均可以得到以下内容：
![实验结果图](http://7xilc8.com1.z0.glb.clouddn.com/tmp2.png)

## 四、扩展研究
根据[Gluster官方文档](http://gluster.readthedocs.io/en/latest/Administrator%20Guide/Setting%20Up%20Clients/)
1）在Linux中实现Native挂载。
2）在Windows 7 企业版中实现NFS挂载。
