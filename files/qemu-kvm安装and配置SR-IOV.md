# qemu-kvm安装and配置桥接和SR-IOV



> kvm和docker的区别：kvm是全虚拟化，需要模拟各种硬件，docker是容器，共享宿主机的CPU，内存，swap等。本文安装的qemu-kvm属于kvm虚拟化，其中：kvm负责cpu虚拟化和内存虚拟化，QEMU模拟IO设备（网卡、磁盘等）。
>
> 参考资料：
>
> qemu和docker区别：https://www.cnblogs.com/boyzgw/p/6807986.html
>
> qemu，kvm，qemu-kvm关系：https://www.cnblogs.com/echo1937/p/7138294.html

## 1. 安装

ubuntu环境安装：

```shell
sudo apt-get install qemu virt-manager qemu-kvm
```

centos环境安装：

```shell
yum install qemu-kvm qemu-img virt-manager libvirt libvirt-python virt-manager libvirt-client virt-install virt-viewer -y
```

其中，virt-manager是虚拟机管理工具，相当于windows环境下的vmware软件。



## 2. 新建虚拟机

下载系统镜像（以ubuntu14.04为例）：

```shell
wget http://mirrors.163.com/ubuntu-releases/14.04/ubuntu-14.04.6-server-amd64.iso
```

创建一个虚拟磁盘, -f指定格式, 文件名kvm.qcow2 ,大小为20G 

```shell
qemu-img create -f qcow2 -o preallocation=metadata ubuntu14_04.qcow2 20G
```

通过命令行安装虚拟机：

```shell
virt-install --virt-type=kvm --name=ubuntu14_04 --vcpus=2 --memory=2048 --location=ubuntu-14.04.6-server-amd64.iso --disk path=ubuntu14_04.qcow2,size=20,format=qcow2 --network network=default --graphics none --extra-args='console=ttyS0' --force
```

| 参数         | 含义                                               |
| ------------ | -------------------------------------------------- |
| --virt-type  | 虚拟化类型                                         |
| --name       | 虚拟机的名字                                       |
| --vcpus      | CPU个数                                            |
| --memory     | 内存大小，单位MB                                   |
| --location   | ISO系统镜像位置                                    |
| --disk path  | 磁盘位置, 大小, 格式等                             |
| --network    | 网络                                               |
| --graphics   | guest显示设置, 设置为none时，表示从命令行安装      |
| --extra-args | 如果从命令行安装，需要指定该参数为 'console=ttyS0' |

安装虚拟机时指定网桥（需要先配置好桥接，方法在下面）：

```shell
virt-install --virt-type=kvm --name=ubuntu14_04 --vcpus=2 --memory=2048 --location=ubuntu-14.04.6-server-amd64.iso --disk path=ubuntu14_04.qcow2,size=20,format=qcow2 --network bridge=br0 --graphics none --extra-args='console=ttyS0' --force
```

安装虚拟机时指定PCI设备（需要先配置好SR-IOV，方法在下面，02:00.1是SR-IOV虚拟出来的网卡的PCI编号）：

```shell
virt-install --virt-type=kvm --name=ubuntu14_04 --vcpus=2 --memory=2048 --location=ubuntu-14.04.6-server-amd64.iso --disk path=ubuntu14_04.qcow2,size=20,format=qcow2 --network network=default --hostdev=02:00.1 --graphics none --extra-args='console=ttyS0' --force
```

其中，hostdev可以是以下几种形式：

> **--hostdev pci_0000_02_00_1**
>
> A node device name via libvirt, as shown by  ```virsh nodedev-list```
>
> **--hostdev 001.003**
>
> USB by bus, device (via lsusb).
>
> **--hostdev 0x1234:0x5678**
>
> USB by vendor, product (via lsusb).
>
> **--hostdev 02.00.1**
>
> PCI device (via lspci).

<font color="red">如果出现错误</font>：

> WARNING /home/user/ubuntu14_04.qcow2 may not be accessible by the hypervisor. You will need to grant the <font color="red">'qemu'</font> user search permissions for the following directories: <font color="red">['/home/user']</font>

出现这种错误是因为qemu用户没有权限访问当前用户的家目录，修改权限为其他用户可以访问当前用户目录即可解决：

```shell
cd /home
chmod 755 user
```

> drwxr-x<font color="red">r-x</font>. 12 user    user    4096 Oct 19 11:43 user

## 3. 使用虚拟机

| 命令                       | 含义                                                         |
| -------------------------- | ------------------------------------------------------------ |
| virsh dumpxml name         | 查看虚拟机配置文件                                           |
| virsh start name           | 启动kvm虚拟机                                                |
| virsh console name         | 连接到虚拟机终端                                             |
| virsh shutdown name        | 正常关机                                                     |
| virsh destroy name         | 非正常关机，相当于物理机直接拔掉电源                         |
| virsh undefine name        | 彻底删除，无法找回，如果想找回来，需要备份/etc/libvirt/qemu的xml文件 |
| virsh define file-name.xml | 根据配置文件定义虚拟机                                       |
| virsh suspend name         | 挂起，终止                                                   |
| virsh resume name          | 恢复挂起状态                                                 |
| virsh edit name            | 编辑虚拟机配置文件*（编辑之后需要重启虚拟机才会生效）*       |
| virsh list                 | 查看正在运行的虚拟机*（在root账号或加sudo运行才能看到）*     |
| virsh list --all           | 查看所有虚拟机*（在root账号或加sudo运行才能看到）*           |

配置桥接网络：

```shell
virsh iface-bridge --interface eth0 --bridge br0
```

eth0是设置桥接的物理网卡名称，br0是桥接网卡的名称。

查看桥接网络：

```shell
brctl show 
virsh iface-list 
```

删除桥接网卡：

```shell
virsh iface-unbridge br0
```

<font color="red">OR</font> 通过修改配置文件配置桥接网络：

```shell
virsh edit ubuntu14_04
```

修改的内容如下：

```xml
    <interface type='network'>              ###这一行修改接口模式为"bridge"
      <mac address='52:54:00:c6:9f:8a'/>
      <source network='default'/>      			###这一行修改源为"bridge='br0'"
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
```



## 4. qemu配置SR-IOV使用RDMA网卡

查看物理机是否开启VT：

```shell
cat /proc/cpuinfo | grep vmx
```

如果输出内容中有 vmx，仅仅说明CPU支持 VT，还需要通过如下命令查看是否开启：

```shell
lsmod |grep kvm 
```

如果已经开启VT，会显示 <font color="red">kvm_intel</font> 和 <font color="red">kvm</font>，如果没开启，需要进BIOS设置。

已经开启的显示示例：

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20201019163429.png" style="zoom:50%;" />

在启动菜单的内核CMDLINE中开启iommu，/boot/grub2/grub.cfg不能直接修改，需通过修改 /etc/default/grub：

```shell
vim /etc/default/grub
```

> GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet <font color="red">intel_iommu=on iommu=pt</font>"

然后更新grub配置：

```shell
grub2-mkconfig -o /boot/grub2/grub.cfg
```

更新配置后，重启。

为RDMA网卡设置SR-IOV之前，确认已安装好网卡驱动。

### 4.1 在固件上开启SR-IOV

1. 运行 mst（以下命令均是在root账户执行）

   ```shell
   mst start
   ```

   > Starting MST (Mellanox Software Tools) driver set
   >
   > Loading MST PCI module - Success
   >
   > Loading MST PCI configuration module - Success
   >
   > Create devices
   >
   > Unloading MST PCI module (unused) - Success

2. 查看PCI插槽中的HCA设备

   ```shell
   mst status
   ```

   > MST modules:
   >
   > \------------
   >
   >   MST PCI module is not loaded
   >
   >   MST PCI configuration module loaded
   >
   > 
   >
   > MST devices:
   >
   > \------------
   >
   > <font color="red">/dev/mst/mt4119_pciconf0</font>     - PCI configuration cycles access.
   >
   > ​                  domain:bus :dev.fn=0000:02:00.0 addr.reg=88 data.reg=92 cr_bar.gw_offset=-1
   >
   > ​                  Chip revision is: 00
   >
   > /dev/mst/mt4119_pciconf1     - PCI configuration cycles access.
   >
   > ​                  domain:bus :dev.fn=0000:81:00.0 addr.reg=88 data.reg=92 cr_bar.gw_offset=-1
   >
   > ​                  Chip revision is: 00

3. 查询设备的SRIOV是否开启、虚拟化数量

   ```shell
   mlxconfig -d /dev/mst/mt4119_pciconf0 q | grep -E "SRIOV_EN|NUM_OF_VFS"
   ```

   > NUM_OF_VFS             0    
   >
   > SRIOV_EN              	 False(0)

4. 开启SR-IOV并设置VFS的数量

   - SRIOV_EN=1;		  开启SRIOV
   - NUM_OF_VFS=4 ;   将VFS数量设置为4

   ```shell
   mlxconfig -d /dev/mst/mt4119_pciconf0 set SRIOV_EN=1 NUM_OF_VFS=4
   ```

5. 查看是否设置成功

   ```shell
   mlxconfig -d /dev/mst/mt4119_pciconf0 q | grep -E "SRIOV_EN|NUM_OF_VFS"
   ```

   > NUM_OF_VFS             4   
   >
   > SRIOV_EN              	 True(1)

   <font color="red">注意</font>：此时，无法通过lspci看到VFS，只有在MLNX OFED驱动程序上启用SR-IOV后，你才能看到它们。

### 4.2 在MLNX_OFED驱动上开启SR-IOV

1. 找到设备，本示例中，有两个设备处于激活动态：mlx5_0对应接口 "ib0"，mlx5_1对应接口 "ib1"，我们只配对 "mlx5_0" 配置。

   ```shell
   ibstat
   ```

   > <font color="red">CA 'mlx5_0'</font>
   >
   > ​	......
   >
   > ​	Port 1:
   >
   > ​		<font color="red">State: Active</font>
   >
   > ​		<font color="red">Physical state: LinkUp</font>
   >
   > ​		......
   >
   > CA 'mlx5_1'
   >
   > ​	......
   >
   > ​	Port 1:
   >
   > ​		State: Active
   >
   > ​		Physical state: LinkUp
   >
   > ​		......

   ```shell
   ibdev2netdev
   ```

   > <font color="red">mlx5_0 port 1 ==> ib0 (Up)</font>
   >
   > mlx5_1 port 1 ==> ib1 (Up)

2. 查看固件中配置的VFS数量

   ```shell
   cat /sys/class/net/ib0/device/sriov_totalvfs
   ```

   > 4

   <font color="red">注意</font>：这是一个查看操作，配置VFS数量应使用上面用到的命令：

   ```shell
   mlxconfig -d /dev/mst/mt4119_pciconf0 set SRIOV_EN=1 NUM_OF_VFS=4
   ```

3. 查看当前设备的VFS数量（三种方式结果一样）

   ```shell
   cat /sys/class/infiniband/mlx5_0/device/mlx5_num_vfs
   cat /sys/class/net/ib0/device/sriov_numvfs
   cat /sys/class/net/ib0/device/mlx5_num_vfs
   ```

   > 0

   <font color="red">注意</font>：如果命令执行失败，可能意味着未加载驱动程序。

   <font color="red">注意</font>：mlx5_num_vfs和sriov_numvfs的区别在于，即使操作系统未加载虚拟化模块（未向grub文件添加intel_iommu=on），mlx5_num_vfs也存在；sriov_numvfs 仅在将intel_iommu=on添加到grub文件时才适用。因此，如果没有sriov_numvfs文件，请检查是否已将Intel_iommu=on添加到grub文件中。

   <font color="red">注意</font>：因内核版本不同，可能没有部分选项。

4. 设置VFS的数量（三种方式，任选其一）

   ```shell
   echo 4 > /sys/class/infiniband/mlx5_0/device/mlx5_num_vfs
   cat /sys/class/infiniband/mlx5_0/device/mlx5_num_vfs
   ```

   ```shell
   echo 4 > /sys/class/net/ib0/device/sriov_numvfs
   cat /sys/class/net/ib0/device/sriov_numvfs
   ```

   ```shell
   echo 4 > /sys/class/net/ib0/device/mlx5_num_vfs
   cat /sys/class/net/ib0/device/mlx5_num_vfs
   ```

   > 4

   <font color="red">如出现错误信息</font>：

   > echo: write error: Cannot allocate memory

   修改 /etc/default/grub并重新生成/boot/grub2/grub.cfg：

   ```shell
   vim /etc/default/grub
   ```

   > GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet intel_iommu=on iommu=pt <font color="red">pci=realloc</font>"

   ```shell
   grub2-mkconfig -o /boot/grub2/grub.cfg
   ```

   更新配置后，重启。

   <font color="red">注意</font>：

   1.更改VFS的数量是临时的，服务器重新启动后，设置的值会丢失。

   2.写入sysfs文件时，适用以下规则：

   - 如果未分配VFS，则VFS的数量可以更改为任何有效值（0-固件设置步骤中设置的最大VFS）；
   - 如果有分配给虚拟机的VFS，则无法更改VFS的数量；
   - 如果在未分配VFS的情况下，管理员在PF上卸载驱动程序，则驱动程序将卸载并禁用SRI-OV；
   - 如果在卸载PF驱动程序时分配了VFS，则不会禁用SR-IOV。这意味着VF在VM上可见，但它们将无法运行。这适用于使用pci_stub而非vfio内核的操作系统。
     - VF驱动程序将发现这种情况并关闭其资源；
     - 重新加载PF上的驱动程序后，VF可以运行。 VF的管理员将需要重新启动驱动程序才能继续使用VF。

5. 查看PCI设备

   ```shell
   lspci -D | grep Mellanox
   ```

   > 0000:02:00.0 Infiniband controller: **Mellanox** Technologies MT27800 Family [ConnectX-5]
   >
   > 0000:02:00.1 Infiniband controller: **Mellanox** Technologies MT27800 Family [ConnectX-5 <font color="red">Virtual Function</font>]
   >
   > 0000:02:00.2 Infiniband controller: **Mellanox** Technologies MT27800 Family [ConnectX-5 <font color="red">Virtual Function</font>]
   >
   > 0000:02:00.3 Infiniband controller: **Mellanox** Technologies MT27800 Family [ConnectX-5 <font color="red">Virtual Function</font>]
   >
   > 0000:02:00.4 Infiniband controller: **Mellanox** Technologies MT27800 Family [ConnectX-5 <font color="red">Virtual Function</font>]
   >
   > 0000:81:00.0 Infiniband controller: **Mellanox** Technologies MT27800 Family [ConnectX-5]

   <font color="red">注意</font>：带 <font color="red">Virtual Function</font> 的四个PCI设备就是通过SR-IOV虚拟化出来的RDMA网卡。

   ```shell
   ibdev2netdev -v
   ```

   > 0000:02:00.0 mlx5_0 (MT4119 - MCX555A-ECAT) CX555A - ConnectX-5 QSFP28 fw 16.26.4012 port 1 (ACTIVE) ==> ib0 (Up)
   >
   > 0000:81:00.0 mlx5_1 (MT4119 - MCX555A-ECAT) CX555A - ConnectX-5 QSFP28 fw 16.27.2008 port 1 (ACTIVE) ==> ib1 (Up)
   >
   > <font color="red">0000:02:00.1 mlx5_2 (MT4120 - NA) fw 16.26.4012 port 1 (DOWN ) ==> ib2 (Down)</font>
   >
   > <font color="red">0000:02:00.2 mlx5_3 (MT4120 - NA) fw 16.26.4012 port 1 (DOWN ) ==> ib3 (Down)</font>
   >
   > <font color="red">0000:02:00.3 mlx5_4 (MT4120 - NA) fw 16.26.4012 port 1 (DOWN ) ==> ib4 (Down)</font>
   >
   > <font color="red">0000:02:00.4 mlx5_5 (MT4120 - NA) fw 16.26.4012 port 1 (DOWN ) ==> ib5 (Down)</font>

   此时，你可以看到PF上有4个VF：

   | PCI 编号     | VF 编号 |
   | ------------ | ------- |
   | 0000:02:00.1 | 0       |
   | 0000:02:00.2 | 1       |
   | 0000:02:00.3 | 2       |
   | 0000:02:00.4 | 3       |

   注意：这个时候，我们看到通过SR-IOV虚拟出来的4个网卡的状态还是DOWN：

   > 0000:02:00.1 mlx5_2 (MT4120 - NA) fw 16.26.4012 port 1 (<font color="red">DOWN</font> ) ==> ib2 (Down)
   >
   > 0000:02:00.2 mlx5_3 (MT4120 - NA) fw 16.26.4012 port 1 (<font color="red">DOWN</font> ) ==> ib3 (Down)
   >
   > 0000:02:00.3 mlx5_4 (MT4120 - NA) fw 16.26.4012 port 1 (<font color="red">DOWN</font> ) ==> ib4 (Down)
   >
   > 0000:02:00.4 mlx5_5 (MT4120 - NA) fw 16.26.4012 port 1 (<font color="red">DOWN</font> ) ==> ib5 (Down)

   可以通过以下命令来查看：

   ```
   ibstatus
   ```

   > ......
   >
   > Infiniband device '<font color="red">mlx5_2</font>' port 1 status:
   >
   > ​	default gid:	 fe80:0000:0000:0000:0000:0000:0000:0000
   >
   > ​	base lid:	 0xffff
   >
   > ​	sm lid:		 0x2
   >
   > ​	state:		 1: <font color="red">DOWN</font>
   >
   > ​	phys state:	 5: LinkUp
   >
   > ​	rate:		 100 Gb/sec (4X EDR)
   >
   > ​	link_layer:	 InfiniBand
   >
   > ......

6. 在ip池查看VFS

   ```shell
   ip link show
   ```

   > ......
   >
   > 4: ib0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2044 qdisc mq state UP mode DEFAULT group default qlen 256
   >
   >   link/infiniband 20:00:0a:12:fe:80:00:00:00:00:00:00:ec:0d:9a:03:00:c0:41:d4 brd 00:ff:ff:ff:ff:12:40:1b:ff:ff:00:00:00:00:00:00:ff:ff:ff:ff
   >
   >   vf 0 MAC 00:00:00:00:00:00, spoof checking off, link-state disable, trust off, query_rss off
   >
   >   vf 1 MAC 00:00:00:00:00:00, spoof checking off, link-state disable, trust off, query_rss off
   >
   >   vf 2 MAC 00:00:00:00:00:00, spoof checking off, link-state disable, trust off, query_rss off
   >
   >   vf 3 MAC 00:00:00:00:00:00, spoof checking off, link-state disable, trust off, query_rss off
   >
   > ......
   
8. 为每个VF设置GUID和PORT

   要添加GUID和PORT，需要<font color="red">先取消绑定该网卡</font>：

   ```shell
   echo 0000:02:00.1 > /sys/bus/pci/drivers/mlx5_core/unbind
   ```

   设置GUID和PORT：

   ```shell
   echo 11:22:33:44:77:66:77:60 > /sys/class/infiniband/mlx5_0/device/sriov/0/node
   echo 11:22:33:44:77:66:77:61 > /sys/class/infiniband/mlx5_0/device/sriov/0/port
   
   cat /sys/class/infiniband/mlx5_0/device/sriov/0/node
   cat /sys/class/infiniband/mlx5_0/device/sriov/0/port
   ```
   
   /sys/class/infiniband/mlx5_0/device/sriov/<font color="red">0</font>/node 中的 <font color="red">0 </font>是VF的编号。
   
   重新绑定该网卡：
   
   ```shell
   echo 0000:02:00.1 > /sys/bus/pci/drivers/mlx5_core/bind
   ```

8. 为每个VF设置管理状态为 "Follow"（设置完成后，状态为<font color="red">DOWN</font>的网卡将会开始初始化，并改为<font color="green">ACTIVE</font>）

   ```shell
   cat /sys/class/infiniband/mlx5_0/device/sriov/0/policy
   ```

   > Down

   ```shell
   echo Follow > /sys/class/infiniband/mlx5_0/device/sriov/0/policy
   cat /sys/class/infiniband/mlx5_0/device/sriov/0/policy
   ```

   > Follow

   验证网卡的端口状态：

   > Infiniband device '<font color="red">mlx5_2</font>' port 1 status:
   >
   > ​	default gid:	 fe80:0000:0000:0000:1122:3344:7766:7761
   >
   > ​	base lid:	 0x1d
   >
   > ​	sm lid:		 0x2
   >
   > ​	state:		 4: <font color="green">ACTIVE</font>
   >
   > ​	phys state:	 5: LinkUp
   >
   > ​	rate:		 100 Gb/sec (4X EDR)
   >
   > ​	link_layer:	 InfiniBand

### 4.3 为qemu添加SR-IOV虚拟化的网卡

1. 查看PCI设备信息

   ```shell
   lshw -c network -businfo
   ```

   > Bus info     Device   Class     Description
   >
   > ========================================================
   >
   > ......
   >
   > pci@<font color="red">0000:02:00.1</font> ib2     network    MT27800 Family [ConnectX-5 Virtual Function]
   >
   > pci@<font color="red">0000:02:00.2</font> ib3     network    MT27800 Family [ConnectX-5 Virtual Function]
   >
   > pci@<font color="red">0000:02:00.3</font> ib4     network    MT27800 Family [ConnectX-5 Virtual Function]
   >
   > pci@<font color="red">0000:02:00.4</font> ib5     network    MT27800 Family [ConnectX-5 Virtual Function]
   >
   > ......

   这一步看到的信息，其实在刚才通过 "ibdev2netdev -v" 命令已经得到了。

2. 将设备从宿主机deattach

   ```shell
   virsh nodedev-detach pci_0000_02_00_1
   ```

   命令中，pci_0000_02_00_1 是根据上面由SR-IOV虚拟化出来的PCI设备编号拼接起来的：

   > <font color="red">0000</font>:<font color="green">02</font>:<font color="blue">00</font>.<font color="orange">1</font>  -->  pci\_<font color="red">0000</font>_<font color="green">02</font>\_<font color="blue">00</font>\_<font color="orange">1</font>

   也可以直接通过如下命令查看：

   ```shell
   virsh nodedev-list --tree | grep pci
   ```

   >  ......
   >
   >  |  +- <font color="red">pci_0000_02_00_1</font>
   >
   >  |  +- pci_0000_02_00_2
   >
   >  ......

   如果该虚拟设备不再被使用，需要在 virt-manager 中首先将该设备移除，然后在主机上重新挂载该设备：

   ```shell
   virsh nodedev-reattach pci_0000_02_00_1
   ```

3. 配置VF直通

   - 方法1（interface）：在devices段落里加入（<font color="red">该方法未成功</font>）

     ```shell
     virsh edit ubuntu14_04
     ```

     内容如下：

     ```xml
     <interface type='hostdev' managed='yes'>
       <mac address='52:54:00:ad:ef:8d'/>
       <source>
         <address type='pci' domain='0x0000' bus='0x02' slot='0x00' function='0x1'/>
       </source>
       <vlan>
         <tag id='4010'/>
       </vlan>
     </interface>
     ```

     如不需要设置mac和vlan，可以去掉相应标签。

     其中，address中的参数是根据 "lshw -c network -businfo" 获得的信息配置的，例如，我要配置的PCI设备编号是：

     > pci@<font color="red">0000</font>:<font color="green">02</font>:<font color="blue">00</font>.<font color="orange">1</font> ib2
     
     注意对应关系，domain: 0x<font color="red">0000</font>, bus: 0x<font color="green">02</font>, slot: 0x<font color="blue">00</font>, function: 0x<font color="orange">1</font>.
   
   - 方法2（hostdev）：在devices段落里加入（本文测试中，该方法有效）
   
     ```xml
     <hostdev mode='subsystem' type='pci' managed='yes'>
       <source>
         <address domain='0x0000' bus='0x02' slot='0x00' function='0x1'/>
       </source>
      </hostdev>
     ```
     
   - <font color="red">方法选择</font>：
     - 方法1：功能多，可以配置mac和vlan；
     - 方法2：mac和vlan需要自己在宿主上敲ip命令设置。
   
4. 连接虚拟机，验证是否有RDMA网卡

    ```shell
     lspci | grep Mellanox
    ```

   > 00:06.0 Infiniband controller: **Mellanox** Technologies MT28800 Family [ConnectX-5 Virtual Function]
   
   可以看到，在虚拟机中有RDMA网卡，接下来就是安装驱动等操作了。
   
   > 安装驱动可参考：[https://www.cnblogs.com/sctb/p/13179542.html](https://www.cnblogs.com/sctb/p/13179542.html)
   
   安装驱动后，通过以下命令查看网卡状态：
   
   ```
   ibstatus
   ```
   
   > Infiniband device 'mlx5_0' port 1 status:
   >
   > ​	default gid:	 fe80:0000:0000:0000:1122:3344:7766:7761
   >
   > ​	base lid:	 0x1d
   >
   > ​	sm lid:		 0x2
   >
   > ​	state:		 4: <font color="green">ACTIVE</font>
   >
   > ​	phys state:	 5: LinkUp
   >
   > ​	rate:		 100 Gb/sec (4X EDR)
   >
   > ​	link_layer:	 InfiniBand
   
5. 每次重启之后需要进行的操作：

    ```shell
    echo 4 > /sys/class/infiniband/mlx5_0/device/mlx5_num_vfs
    ibdev2netdev -v
    
    echo 0000:02:00.1 > /sys/bus/pci/drivers/mlx5_core/unbind
    
    echo 11:22:33:44:77:66:77:60 > /sys/class/infiniband/mlx5_0/device/sriov/0/node
    echo 11:22:33:44:77:66:77:61 > /sys/class/infiniband/mlx5_0/device/sriov/0/port
    
    cat /sys/class/infiniband/mlx5_0/device/sriov/0/node
    cat /sys/class/infiniband/mlx5_0/device/sriov/0/port
    
    echo 0000:02:00.1 > /sys/bus/pci/drivers/mlx5_core/bind
    
    cat /sys/class/infiniband/mlx5_0/device/sriov/0/policy
    
    echo Follow > /sys/class/infiniband/mlx5_0/device/sriov/0/policy
    cat /sys/class/infiniband/mlx5_0/device/sriov/0/policy
    ```

    <font color="red">注意</font>：为多个虚拟虚拟网卡配置的node和port不能一样，不然会一直处于 <font color="blue">INIT</font> 状态，不能变成 <font color="green">ACTIVE</font> 状态！



> RDMA配置SR-IOV参考资料：https://community.mellanox.com/s/article/howto-configure-sr-iov-for-connect-ib-connectx-4-with-kvm--infiniband-x
>
> 另一个资料，缺少activate步骤：https://community.mellanox.com/s/article/HowTo-Configure-SR-IOV-for-ConnectX-4-ConnectX-5-ConnectX-6-with-KVM-Ethernet

