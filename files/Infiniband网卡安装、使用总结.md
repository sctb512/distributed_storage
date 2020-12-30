# Infiniband网卡安装、使用总结

> 最近多次安装、使用infiniband网卡，每次都要到处寻找相关资料，所以决定做此总结，方便查找。



## 1. 基础知识

首先，得了解什么是RDMA，贴几个资料：

[深入浅出全面解析RDMA](https://tjcug.github.io/blog/2018/06/04/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BA%E5%85%A8%E9%9D%A2%E8%A7%A3%E6%9E%90RDMA/)

[RDMA技术详解（一）：RDMA概述](https://zhuanlan.zhihu.com/p/55142557)

[RDMA技术详解（二）：RDMA Send Receive操作](https://zhuanlan.zhihu.com/p/55142547)

然后得了解如何实现，这两个可以有个初步了解：

[RDMA编程：事件通知机制](https://www.jianshu.com/p/4d71f1c8e77c)

[RDMA read and write with IB verbs](https://thegeekinthecorner.wordpress.com/2010/09/28/rdma-read-and-write-with-ib-verbs/)

编程过程，真正有用的还是官方的手册：

[RDMA Aware Networks Programming User Manual](https://www.mellanox.com/sites/default/files/related-docs/prod_software/RDMA_Aware_Programming_user_manual.pdf)

mellanox官方社区能找到很多你需要的东西：

[https://community.mellanox.com/s/](https://community.mellanox.com/s/)

也下了个中文版，但我感觉英文版看着更好。中文版下载：

百度云: [https://pan.baidu.com/s/1BkbinPMy6fwN7J5BPFadDw](https://pan.baidu.com/s/1BkbinPMy6fwN7J5BPFadDw) 提取码: rm8i

蓝奏云：https://wwa.lanzous.com/iXUd6jm7qla 密码: 4aps

RDMA编程入门可参考的项目：

[https://github.com/tarickb/the-geek-in-the-corner](https://github.com/tarickb/the-geek-in-the-corner)

[https://github.com/jcxue/RDMA-Tutorial](https://github.com/jcxue/RDMA-Tutorial)



## 2. 驱动安装

1. 下载驱动，进入网站选择相应系统和软件版本，archive versions这里可以下载旧版本驱动

   [http://www.mellanox.com/page/software_overview_ib ](http://www.mellanox.com/page/software_overview_ib )

   <img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20200622212026.png" style="zoom:50%;" />

   ubuntu16.04平台5.0-2.1.8.0的下载链接为：

   ```shell
   wget http://content.mellanox.com/ofed/MLNX_OFED-5.0-2.1.8.0/MLNX_OFED_LINUX-5.0-2.1.8.0-ubuntu16.04-x86_64.iso
   ```

   版本5.1之后链接细微变化，ubuntu18.04平台5.1-2.5.8.0的下载链接为：

   ```shell
   wget https://www.mellanox.com/downloads/ofed/MLNX_OFED-5.1-2.5.8.0/MLNX_OFED_LINUX-5.1-2.5.8.0-ubuntu18.04-x86_64.iso
   ```

   

   其它平台和版本的驱动，可以自己修改。

2. 挂载或解压，如果下载的iso则挂载，若是tgz就解压，下面是挂载命令：

   ```shell
   sudo mount -o ro,loop MLNX_OFED_LINUX-5.0-2.1.8.0-ubuntu16.04-x86_64.iso /mnt
   ```

3. 安装

   ```shell
   cd /mnt
   sudo ./mlnxofedinstall
   ```

   可能会提示你安装一堆东西，复制，安装就可以了。

   安装成功截图：

   <img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20200622222008.png" style="zoom:50%;" />

4. 执行以下命令：

   ```shell
   sudo /etc/init.d/openibd restart
   sudo /etc/init.d/opensmd restart
   ```
   
5. 查看网卡状态：

   ```shell
   sudo hca_self_test.ofed
   ```

   <img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20200622215258.png" style="zoom:50%;" />

   没有 failed 就对了。

   一些其它查看网卡信息的命令：

   ```
   ibstat
   ibstatus
   ibv_devinfo
   ibv_devices	#查看本主机的infiniband设备
   ibnodes	#查看网络中的infiniband设备
   ```

6. 配置ip

   - ubuntu执行：

     ```shell
     sudo vim /etc/network/interfaces
     ```

     在文件中添加如下内容：

     ```
     auto enp1s0
     iface enp1s0 inet static
     address 172.16.0.104
     netmask 255.255.255.0
     broadcast 172.16.0.255
     ```

     enp1s0是网卡名称，通过ifconfig查看，address是要给infiniband网卡配置的ip地址。

     重启网络服务：

     ```
     sudo service networking restart
     ```

     

   - centos执行：

     ```
     sudo vim /etc/sysconfig/network-scripts/ifcfg-ib0
     ```

     添加如下内容：

     ```
     DEVICE=ib0
     BOOTPROTO=static
     IPADDR=172.16.0.104
     NETMASK=255.255.255.0
     BROADCAST=172.16.0.255
     NETWORK=172.16.0.0
     ONBOOT=yes
     ```

     重启网口：

     ```
     sudo ifdown ib0
     sudo ifup ib0
     ```

     

## 3. 性能测试

1. 服务端运行：

   ```
   ib_send_bw -a -c UD -d mlx4_0 -i 1
   ```

   注意，参数 -i 指定端口，在一个网卡有多个网口的时候，需要指定测试的端口，具体哪个端口，通过 ibstatus 可以看到。

2. 客户端运行：

   ```
   ib_send_bw -a -c UD -d mlx4_0 -i 1 172.16.0.102
   ```

   最后面的ip地址是服务端infiniband网卡的ip地址。

   <img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20200622221446.png" style="zoom:50%;" />

   3. 其他测试项

      ```
      ib_atomic_bw   ib_atomic_lat  ib_read_bw     ib_read_lat    ib_send_bw     ib_send_lat    ib_write_bw    ib_write_lat
      ```

      bw表示测试带宽，lat表示测试延迟，参数同上，可以i通过 --help 查看。

      

## 4. 其他问题

更换网卡工作模式：

有些网卡，当你安装好驱动后，通过 ibstatus 命令，会出现下面的情况：

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20200622221842.png" style="zoom:50%;" />

可以看到，该网卡现在处于 Ethernet 的工作模式，如果想要切换成infiniband模式，参考如下链接：

[https://community.mellanox.com/s/article/howto-change-port-type-in-mellanox-connectx-3-adapter](https://community.mellanox.com/s/article/howto-change-port-type-in-mellanox-connectx-3-adapter)

查看当前工作模式：

```
sudo /sbin/connectx_port_config -s
```

输入以下命令切换工作模式：

```
sudo /sbin/connectx_port_config
```

<img src="https://gitee.com/sctb/abin_pictures/raw/master/imgs/20200622222510.png" style="zoom:50%;" />

如果提示如图，说明不支持infiniband模式，否则，就切换成功了，再次使用一下命令可以验证：

```
sudo /sbin/connectx_port_config -s
```



不能切换到infiniband工作模式，并不代表不支持RDMA，处于Ethernet模式的网卡使用 RoCE 协议工作。

> RDMA 协议：底层可以是以太网（ RoCE 或者 iWARP ）或者 Infiniband



有些网卡只支持Ethernet（RoCE），不支持Infiniband模式，也就是想从Ethernet切换到Infiniband模式时不能成功，这个要提前了解好。我目前了解到的，Connectx-3只支持Ethernet模式。

[https://community.mellanox.com/s/question/0D51T00006RVtsz/connectx4-says-it-doesnt-support-linktypep1-configuration](https://community.mellanox.com/s/question/0D51T00006RVtsz/connectx4-says-it-doesnt-support-linktypep1-configuration)

