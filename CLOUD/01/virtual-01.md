# 云计算基础 -- 虚拟化技术

## Linux虚拟化技术

#### 常用虚拟化技术

  vmware（收费，企业版 esxi ）
  https://www.proxmox.com/en/proxmox-ve
  redhat kvm rhev

#### 虚拟化平台

1、查看是否支持虚拟化

```shell 
[root@localhost ~]# grep -P "vmx|svm" /proc/cpuinfo
flags		: ... ... vmx
[root@localhost ~]# lsmod |grep kvm
kvm_intel             174841  6 
kvm                   578518  1 kvm_intel
irqbypass              13503  1 kvm
```

2、创建虚拟机 2cpu，4G内存（默认用户名: root  密码: a）

```shell
[root@localhost ~]# base-vm create ecs
vm ecs create                                              [  OK  ]
[root@localhost ~]# 
```

3、验证 yum 仓库的配置

```shell
[root@localhost ~]# yum makecache
Loaded plugins: fastestmirror
Determining fastest mirrors
local_repo                                          	    | 3.6 kB   00:00     
(1/4): local_repo/group_gz                         	        | 166 kB   00:00     
(2/4): local_repo/filelists_db                      	    | 6.9 MB   00:00     
(3/4): local_repo/primary_db                               	| 5.9 MB   00:00     
(4/4): local_repo/other_db                                 	| 2.5 MB   00:00     
Metadata Cache Created
[root@localhost ~]# yum repolist
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
repo id                           repo name                               status
local_repo                        CentOS-7 - Base                         9,911
repolist: 9,911
[root@localhost ~]#
```

4、安装 libvirtd

```shell
[root@localhost ~]# yum install -y qemu-kvm \
                                   libvirt-daemon \
                                   libvirt-daemon-driver-qemu \
                                   libvirt-client
[root@localhost ~]# systemctl enable --now libvirtd
[root@localhost ~]# virsh version
```

**虚拟机组成**
​    硬盘文件  /var/lib/libvirt/images/
​    配置文件  /etc/libvirt/qemu/

#### 虚拟化实验图例

```mermaid
graph TB
  subgraph <font color=#ff0000>真机</font>
      subgraph linux
        style linux color:#ff0000,fill:#11aaff
        H1[(虚拟机)] & H2[(虚拟机)] & H3[(虚拟机)] --> B{{虚拟网桥 <font color=#ff0000>vbr</font>}} --> E([eth0])
      end
      E --> W(外部网络)
  end
```

#### Linux虚拟机

###### 虚拟机硬盘磁盘文件

###### COW图例

```mermaid
flowchart LR
U2((用户)) -..->|读操作| X2
U2((用户)) -..->|读修改过的数据| X3
U1((用户)) --->|写操作| X3
subgraph D1[原始盘]
  X0([数据块])
  X1([数据块])
end
subgraph D2[前端盘]
  X2([如果数据块不存在])
  X3([数据块副本])
end
X1 --->|写时拷贝副本| X3
X2 -.->|读取原始盘数据| X0
classDef mydisk fill:#ffffc0,color:#ff00ff
class D1,D2 mydisk
classDef X2 fill:#ccf,stroke:#f66,stroke-width:2px,stroke-dasharray: 10, 5
class X2 X2
classDef mydata fill:#0000ff,color:#ffff00
class X0,X1 mydata
classDef X3 fill:#ccffbb,color:#000000
class X3 X3
classDef U1 fill:#ffffff,color:#000000,stroke:#555555,stroke-width:4px;
class U1,U2 U1
```

上传 cirros.qcow2 到虚拟机
通过 qemu-img 创建虚拟机磁盘
命令格式: qemu-img  子命令  子命令参数  虚拟机磁盘文件  大小

```shell
[root@localhost ~]# cp cirros.qcow2 /var/lib/libvirt/images/
[root@localhost ~]# cd /var/lib/libvirt/images/
[root@localhost ~]# qemu-img create -f qcow2 -b cirros.qcow2 vmhost.img 30G
[root@localhost ~]# qemu-img info vmhost.img #查看信息
```

###### 虚拟网络配置

虚拟网络管理命令

| 命令                   | 说明                    |
| ---------------------- | -----------------------|
| virsh net-list [--all] | 列出虚拟网络|
| virsh net-start        | 启动虚拟交换机|
| virsh net-destroy      | 强制停止虚拟交换机|
| virsh net-define       | 根据xml文件创建虚拟网络|
| virsh net-undefine     | 删除一个虚拟网络设备|
| virsh net-edit         | 修改虚拟交换机的配置|
| virsh net-autostart    | 设置开机自启动|

创建配置文件 /etc/libvirt/qemu/networks/vbr.xml

```shell
[root@localhost ~]# vim /etc/libvirt/qemu/networks/vbr.xml
<network>
  <name>vbr</name>
  <forward mode='nat'/>
  <bridge name='vbr' stp='on' delay='0'/>
  <ip address='192.168.100.254' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.100.100' end='192.168.100.200'/>
    </dhcp>
  </ip>
</network>
```

创建虚拟交换机

```shell
[root@localhost ~]# cd /etc/libvirt/qemu/networks/
[root@localhost ~]# virsh net-define vbr.xml
[root@localhost ~]# virsh net-start vbr
[root@localhost ~]# virsh net-autostart vbr
[root@localhost ~]# ifconfig # 查看验证
```

###### 虚拟机管理命令

|命令|说明|
|----|----|
|virsh list [--all]|列出虚拟机|
|virsh start/shutdown|启动/关闭虚拟机|
|virsh destroy|强制停止虚拟机|
|virsh define/undefine|创建/删除虚拟机|
|virsh ttyconsole|显示终端设备|
|virsh console|连接虚拟机的 console|
|virsh edit|修改虚拟机的配置|
|virsh autostart|设置虚拟机自启动|
|virsh dominfo|查看虚拟机摘要信息|
|virsh domiflist|查看虚拟机网卡信息|
|virsh domblklist|查看虚拟机硬盘信息|


###### 虚拟机配置文件

官方文档地址 https://libvirt.org/format.html

1、拷贝 node_base.xml 到虚拟机中

2、拷贝 node_base.xml 到 /etc/libvirt/qemu/虚拟机名字.xml

3、修改配置文件，启动运行虚拟机

```shell
[root@localhost ~]# cp node_base.xml /etc/libvirt/qemu/vmhost.xml
[root@localhost ~]# vim /etc/libvirt/qemu/vmhost.xml
2:	<name>vmhost</name>
3:	<memory unit='KB'>1024000</memory>
4:	<currentMemory unit='KB'>1024000</currentMemory>
5:	<vcpu placement='static'>2</vcpu>
26:	<source file='/var/lib/libvirt/images/vmhost.img'/>
```

###### 创建虚拟机

```shell
[root@localhost ~]# virsh list
[root@localhost ~]# virsh define /etc/libvirt/qemu/vmhost.xml
[root@localhost ~]# virsh start vmhost
[root@localhost ~]# virsh console vmhost # 两次回车
退出使用 ctrl + ]
```

## 公有云简介

#### 常用终端管理工具

###### xshell 使用技巧

使用 lrzsz 上传下载文件

安装软件 

```shell
[root@localhost ~]# yum install lrzsz
```

配置 xshell 激活 zmodem

退出重新登录以后，即可，上传(rz),下载(sz)