# infiniswap安装

环境：ubuntu14.04，内核4.04

```shell
uname -a
```

> Linux ubuntu 4.4.0-142-generic #168~14.04.1-Ubuntu SMP Sat Jan 19 11:26:28 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux

克隆项目：

```shell
git clone https://github.com/SymbioticLab/Infiniswap.git
```



## 1. 客户端守护进程安装



1. 客户端编译安装：

   ```shell
   cd setup
   ./install daemon
   ```

4. 运行客户端守护程序：

   ```shell
   cd ../infiniswap_daemon
   ./infiniswap-daemon 192.168.2.221 9400
   ```

   ip地址 192.168.2.221 是客户端所在主机 ib网卡的地址。



## 2. 服务端块设备（交换分区）安装



1. 进入 setup 目录，修改 install.h

   ```shell
   cd setup
   vim install.h
   ```

2. 修改内容主要是以下两个：

   > \#have the kernel patch for lookup_bdev()
   >
   > \#(HAVE_LOOKUP_BDEV_PATCH), default is undefined
   >
   > have_lookup_bdev_patch=<font color="red">1</font>

   > \#name of physical backup disk
   >
   > \#(BACKUP_DISK), default is "/dev/sda4"
   >
   > backup_disk="<font color="red">/dev/vdb</font>"

   如果以上两项未修改正确，无法编译通过。

3. 服务端编译生成内核模块：

   ```shell
   cd setup
   ./install bd
   ```

4. 服务端配置 portal.list

   ```shell
   vim portal.list
   ```

   > 1
   >
   > 192.168.2.221:9400

   第一行为daemon的数量，后面每行为一个daemon的 ip:port

5. 服务端安装块设备，设置为交换分区：

   ```shell
   ./infiniswap_bd_setup.sh
   
   ```
   
   如果执行 `./infiniswap_bd_setup.sh`出错，可以一步一步执行，该文件内容如下：
   
   ```shell
      modprobe infiniswap 
      mount -t configfs none /sys/kernel/config
      
      nbdxadm -o create_host -i 0 -p $PWD/portal.list #portal.list
      nbdxadm -o create_device -i 0 -d 0
      
      ls /dev/infiniswap0
      mkswap /dev/infiniswap0
      swapon /dev/infiniswap0
   ```
   
   成功挂载为交换分区后，打印的内核日志：
   
   > [  94.543854] IS_create_configfs_files
   >
   > [  99.157269] IS_session_make_group, name=infiniswaphost0
   >
   > [  99.157330] portal_attr_store, buf=1,192.168.2.221:9400,
   >
   > [  99.157335] In IS_session_create() with portal: rdma://1,192.168.2.221:9400,
   >
   > [  99.157527] rdma://1,192.168.2.221:9400,
   >
   > [  99.157879] portal: 192.168.2.221, 9400
   >
   > [  99.158040] IS_create_conn with cpu: 0
   >
   > [  99.158042] IS_create_conn with cpu: 1
   >
   > [ 102.866554] IS_device_make_group, name=infiniswap0
   >
   > [ 102.866612] device_attr_store
   >
   > [ 102.866616] In IS_create_device(), dev_name:infiniswap0
   >
   > [ 102.866619] IS: st_size = 12884901888
   >
   > [ 102.866621] IS_register_block_device
   >
   > [ 102.867318] IS_init_hctx called index=0 xq=ffff880079327e80
   >
   > [ 102.867323] IS_init_hctx called index=1 xq=ffff880079327e98
   >
   > [ 102.867349] IS_register_block_device, dev_name infiniswap0
   >
   > [ 102.867543] IS: init done
   >
   > [ 102.868987] stackbd: init done
   >
   > [ 102.868995] Opened /dev/vdb
   >
   > [ 102.869010] stackbd: Device real capacity: 29360128
   >
   > [ 102.869013] stackbd: Max sectors: 8
   >
   > [ 102.869053] stackbd: done initializing successfully
   >
   > [ 112.873800] rdma_trigger
   >
   > [ 211.265717] kernel_cb_init, created cm_id ffff88003633a000
   >
   > [ 211.265842] cma_event type 0 cma_id ffff88003633a000 (parent)
   >
   > [ 211.266119] cma_event type 2 cma_id ffff88003633a000 (parent)
   >
   > [ 211.266161] rdma_resolve_addr - rdma_resolve_route successful
   >
   > [ 211.266641] created pd ffff880078b8b5c0
   >
   > [ 211.267343] created cq ffff880036338800
   >
   > [ 211.269382] created qp ffff880036338c00
   >
   > [ 211.269388] IS: IS_setup_buffers called on cb ffff88007a9b5000
   >
   > [ 211.269391] IS: size of IS_rdma_info 392
   >
   > [ 211.269394] IS: cb->mem=1 
   >
   > [ 211.269396] IS: IS_setup_buffers, in cb->mem==DMA 
   >
   > [ 211.269619] IS: allocated & registered buffers...
   >
   > [ 211.282962] cma_event type 9 cma_id ffff88003633a000 (parent)
   >
   > [ 211.282974] ESTABLISHED
   >
   > [ 211.282990] rdma_connect successful
   >
   > [ 211.285548] IS_ctx_dma_setup, setup_ctx_dma
   >
   > [ 211.285734] evict_handler, waiting for STOP msg
   >
   > [ 211.463865] Received rkey 80709 addr 7fd423fff000 from peer
   >
   > [ 211.517175] Received rkey 80608 addr 7fd3e3ffd000 from peer
   >
   > [ 239.423567] Adding 12582908k swap on /dev/infiniswap0. Priority:-1 extents:1 across:12582908k SSFS

6. 在Infiniswap目录编写了服务端启动脚本 `r_bd.sh`：

   ```shell
   #!/bin/bash
   
   pwd=111111
   swaps='/dev/vda5'
   
   cd setup
   echo $pwd | sudo -S swapon -s
   
   for i in $swaps; do
       echo $pwd | sudo -S swapoff $i
   done
   
   ./infiniswap_bd_setup.sh
   echo $pwd | sudo -S swapon -s
   ```
   
   其中， swaps的值 ` /dev/vda5` 使通过 `swapon -s`查看到的交换分区名字，可设置多个，如：` '/dev/vda5 /dev/vda6' `，pwd 是登录账号的密码。
   
   添加可执行权限：
   
   ```shell
   chmod +x r_bd.sh
   ```
   
   运行bd：
   
   ```shell
   ./r_bd.sh
   ```

   