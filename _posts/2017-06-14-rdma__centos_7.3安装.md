# rdma centos 7.3安装

## 1、安装依赖包
	yum install epel-release -y  
	yum install gcc gcc-c++ bc openssl-devel automake ncurses-devel libibverbs -y  
    yum install libibverbs-devel libibverbs-utils librdmacm librdmacm-devel librdmacm-utils perl-Switch elfutils-libelf-devel  -y

## 2、 librxe-dev 和 rxe-dev下载
下载地址  
<pre>
Github: https://github.com/SoftRoCE/rxe-dev.git  
Github: https://github.com/SoftRoCE/librxe-dev.git  
</pre>
备注：rxe-dev下载v18版本，即rxe-dev-rxe_submission_v18

## 3、安装rxe-dev
<pre>
unzip rxe-dev-rxe_submission_v18.zip
cd rxe-dev-rxe_submission_v18/
cp /boot/config-3.10.0-514.el7.x86_64 .config
</pre>
备注：使用root用户，执行以下命令
<pre>
make menuconfig</pre>

会出现选择界面（如果没出现，需要安装 ncurse-devel）  
输入 "/" ，然后输入 rxe，按下 enter，会查找有关 rxe 的选择项。  
输入数字 1，就会选择到“Software RDMA over Ethernet (ROCE) driver”的设置，输入 "M" ，选中 RDMA 的配置，如果 输不了 M，那就输入空格。  
移动到保存按钮，回车，装保存到.config中，退出安装界面（exit）。  
然后 vi .config 来确认   
```
CONFIG_RDMA_RXE 为 m,  
CONFIG_INFINIBAND_ADDR_TRANS 和 CONFIG_INFINIBAND_ADDR_TRANS_CONFIGFS 为 y  
```
<pre>
make -j
make modules_install ，可能执行中途 会提示 丢失一些 module，这个 没关系，无关紧要。  
make install  
make headers_install INSTALL_HDR_PATH=/usr  
</pre>
确认 新的内核是否在 grub 引导中。查看 /etc/grub.cfg 即可看见。在开机的时候可以选择 新内核启动  

## 4、安装 librxe-dev
<pre>
cd librxe-dev  
./configure --libdir=/usr/lib64/ --prefix=  
make  
make install
</pre>
重启操作系统，在开机启动时，选择4.7.0-rc3内核  
启动后，查看内核版本  
<pre>
uname -r
</pre>

## 5、验证 rdma
<pre>
[root@aboss ~]# rxe_cfg start
  Name        Link  Driver  Speed  NMTU  IPv4_addr  RDEV  RMTU  
  ens33       yes   e1000                                       
  virbr0      no    bridge                                      
  virbr0-nic  no    tun                                         
[root@aboss ~]# rxe_cfg add ens33
[root@aboss ~]# rxe_cfg status
  Name        Link  Driver  Speed  NMTU  IPv4_addr  RDEV  RMTU          
  ens33       yes   e1000                           rxe0  1024  (3)  
  virbr0      no    bridge                                              
  virbr0-nic  no    tun
</pre>
查看rxe设备  
ibv_devices 程序显示该系统中目前所有设备，而 ibv_devinfo 命令会给出每个设备的具体信息
<pre>
[root@aboss ~]# ibv_devices
    device          	   node GUID
    ------          	----------------
    rxe0            	020c29fffe55c818
[root@aboss ~]# ibv_devinfo  rxe0
hca_id:	rxe0
	transport:			InfiniBand (0)
	fw_ver:				0.0.0
	node_guid:			020c:29ff:fe55:c818
	sys_image_guid:			0000:0000:0000:0000
	vendor_id:			0x0000
	vendor_part_id:			0
	hw_ver:				0x0
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		1024 (3)
			sm_lid:			0
			port_lid:		0
			port_lmc:		0x00
			link_layer:		Ethernet
</pre>

## 6、softRoCE连通性测试
服务端
<pre>
rping -s -a 192.168.1.133 -v -C 10
</pre>
客户端
<pre>
rping -c -a 192.168.1.133 -v -C 10
</pre>

## 7、关于librdmacm编译说明
<pre>
git clone https://github.com/ofiwg/librdmacm.git
cd librdmacm
yum install autoconf automake gettext gettext-devel libtool -y
./autogen.sh
./configure
make
make install
</pre>
## 8、常见问题
(1)如果你克隆虚机，需要解决网卡问题  
(2)使用rdma，请将防火墙与selinx关闭
