---
title: Use RDMA Emulation in Software Using SoftiWARP
---
##### 环境
```
lsb_release -d
Ubuntu 16.04.3 LTS
uname -sr
Linux 4.8.13 
```
`SoftiWARP可以支持4.8.x版本内核`([zrlio/softiwarp#6](https://github.com/zrlio/softiwarp/issues/6))

##### 安装依赖
```
sudo add-apt-repository ppa:linux-rdma/rdma-core-daily
sudo apt update
sudo apt install libncurses5-dev libtool autoconf automake
sudo apt install libnl-3-dev libnl-route-3-dev 
sudo apt install ibverbs-utils
```

##### 添加规则文件
###### /etc/udev/rules.d/40-ib.rules  
```
####  /etc/udev/rules.d/40-ib.rules ####  
KERNEL=="umad*", NAME="infiniband/%k"
KERNEL=="issm*", NAME="infiniband/%k"
KERNEL=="ucm*", NAME="infiniband/%k", MODE="0666"
KERNEL=="uverbs*", NAME="infiniband/%k", MODE="0666"
KERNEL=="uat", NAME="infiniband/%k", MODE="0666"
KERNEL=="ucma", NAME="infiniband/%k", MODE="0666"
KERNEL=="rdma_cm", NAME="infiniband/%k", MODE="0666"
```

##### 下载源码
###### SoftiWARP
```
git clone https://github.com/zrlio/softiwarp.git
```

##### 编译并安装用户库
```
cd softiwarp/userlib/  
./autogen.sh  
./configure 
make 
sudo make install  
sudo ln -s /usr/local/etc/libibverbs.d /etc/libibverbs.d
sudo cp softiwarp/userlib/src/.libs/libsiw-rdmav2.so /lib/x86_64-linux-gnu/  
sudo ldconfig 
```

##### 编译内核模块  
```
cd softiwarp/kernel/  
make  
```

##### 载入内核模块
```
sudo modprobe rdma_cm  
sudo modprobe ib_uverbs  
sudo modprobe rdma_ucm  
sudo mkdir /lib/modules/4.8.x/extra  
sudo cp siw.ko /lib/modules/4.8.x/extra  
sudo insmod /lib/modules/4.8.x/extra/siw.ko  
sudo depmod   
sudo modprobe siw
```

##### 查看模块
```
lsmod | grep rdma
```
显示如下

```
rdma_ucm               28672  0
ib_uverbs              65536  1 rdma_ucm
rdma_cm                53248  1 rdma_ucm
iw_cm                  45056  1 rdma_cm
ib_cm                  45056  1 rdma_cm
ib_core               204800  6 ib_cm,rdma_cm,ib_uverbs,iw_cm,rdma_ucm,siw
configfs               36864  2 rdma_cm
```

##### 查看RDMA设备相关信息
###### userspace verbs模块
```
ls /dev/infiniband/  
```
显示如下

```
rdma_cm  uverbs0  uverbs1
```

###### userspace可使用的RDMA设备
```
ibv_devices  
```
显示如下

```
    device          	   node GUID
    ------          	----------------
    siw_enp0s25     	3417eb9bf6d00000
    siw_lo          	7369775f6c6f0000
```

###### userspace可使用的RDMA设备的相关信息
```
ibv_devinfo
```
显示如下

```
hca_id:	siw_enp0s25
	transport:			iWARP (1)
	fw_ver:				0.0.0
	node_guid:			3417:eb9b:f6d0:0000
	sys_image_guid:			3417:eb9b:f6d0:0000
	vendor_id:			0x626d74
	vendor_part_id:			0
	hw_ver:				0x0
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		1024 (3)
			active_mtu:		1024 (3)
			sm_lid:			0
			port_lid:		0
			port_lmc:		0x00
			link_layer:		Ethernet

hca_id:	siw_lo
	transport:			iWARP (1)
	fw_ver:				0.0.0
	node_guid:			7369:775f:6c6f:0000
	sys_image_guid:			0000:0000:0000:0000
	vendor_id:			0x626d74
	vendor_part_id:			0
	hw_ver:				0x0
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		4096 (5)
			sm_lid:			0
			port_lid:		0
			port_lmc:		0x00
			link_layer:		Ethernet
```

##### 测试
输入

```
rping -s
```
开启另一个终端，输入

```
rping -c -a 127.0.0.1 -C 10 -v
```
显示如下

```
ping data: rdma-ping-0: ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqr
ping data: rdma-ping-1: BCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrs
ping data: rdma-ping-2: CDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrst
ping data: rdma-ping-3: DEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstu
ping data: rdma-ping-4: EFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuv
ping data: rdma-ping-5: FGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvw
ping data: rdma-ping-6: GHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwx
ping data: rdma-ping-7: HIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxy
ping data: rdma-ping-8: IJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxyz
ping data: rdma-ping-9: JKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxyzA
client DISCONNECT EVENT...
```



