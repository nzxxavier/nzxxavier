# 1.配置网络
因为我的All in one有1个管理口和4个网卡扩展出来的2.5G网口，所以在我的pve中会规划3个虚拟交换机
* vmbr0(用于PVE的管理网络)
  * 管理口
* vmbr1(用于PVE虚拟机的网络)
  * 网口2
  * 网口3
  * 网口4
![vmbr1](https://upload-images.jianshu.io/upload_images/12844614-8f208680098539de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* wlan(仅有OpenWrt会连接这个网络用于拨号)
  * 网口1
  * 这里的VLAN不勾选的话OpenWrt拨号不会成功
![wlan](https://upload-images.jianshu.io/upload_images/12844614-e7dc737033aaa502.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 2.安装ImmortalWrt
## 2.1.下载并上传
1. https://downloads.immortalwrt.org/releases/24.10.3/targets/x86/64/immortalwrt-24.10.3-x86-64-generic-ext4-combined-efi.qcow2.gz
qcow2格式为PVE的硬盘格式
2. 解压immortalwrt-24.10.3-x86-64-generic-ext4-combined-efi.qcow2.gz
3. 上传
![上传](https://upload-images.jianshu.io/upload_images/12844614-9532a4b4e71794a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.2.添加虚拟机
1. 创建虚拟机
![创建虚拟机](https://upload-images.jianshu.io/upload_images/12844614-1efd57d0fba44587.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 配置虚拟机，不要配置镜像和磁盘，不配置网络
![常规](https://upload-images.jianshu.io/upload_images/12844614-df47bd56bc64a885.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![操作系统](https://upload-images.jianshu.io/upload_images/12844614-461bb6a808198bcd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![系统](https://upload-images.jianshu.io/upload_images/12844614-18509791e97cb3e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![磁盘](https://upload-images.jianshu.io/upload_images/12844614-7fa242253bd4541d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![CPU](https://upload-images.jianshu.io/upload_images/12844614-10fd3eb3b96143ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![内存](https://upload-images.jianshu.io/upload_images/12844614-c3f953eacc6a0ff2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![网络](https://upload-images.jianshu.io/upload_images/12844614-4eeab5454d166907.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. 添加硬盘
![导入硬盘1](https://upload-images.jianshu.io/upload_images/12844614-31e12db3127dd3dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![导入硬盘2](https://upload-images.jianshu.io/upload_images/12844614-94c38c08b73aa7da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
4. 调整磁盘(这里是增量，大概算一下扩到2G)
![调整磁盘](https://upload-images.jianshu.io/upload_images/12844614-5160a46c3458c2b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
5. 添加网络设备
![添加网络设备](https://upload-images.jianshu.io/upload_images/12844614-33cf250919cddc73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![添加wlan](https://upload-images.jianshu.io/upload_images/12844614-2d14114d1ac9ef0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![添加vmbr1](https://upload-images.jianshu.io/upload_images/12844614-13ff066b00b081d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
6. 结果
![结果](https://upload-images.jianshu.io/upload_images/12844614-1587fad5ca0dbe73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
7. 启动虚拟机
## 2.3.配置虚拟机网络
``` shell
vim /etc/config/network
config interface 'lan'
        option device 'eth1'
        option proto 'static'
        option ipaddr '192.168.10.1'
        option netmask '255.255.255.0'
        option ip6assign '60'
```
## 2.4.扩容根目录
1. 浏览器访问192.168.10.1
2. 修改密码
3. 使用上一步修改的密码ssh登录ImmortalWrt
4. 安装需要使用的软件包
``` shell
opkg update
opkg install cfdisk tune2fs resize2fs
cfdisk 
```
5. cfdisk调整磁盘分区，cfdisk，选择挂载点为/的磁盘，光标选择Resize并回车，接着再回车一次（自动分配剩余空间），光标选择Write并回车，光标选择Quit并回车
6. 修改容量后需要手动对齐一下分区，记得把/dev/sda2改成步骤5里那个磁盘分区的名字
``` shell
mount -o remount,ro /
tune2fs -O^resize_inode /dev/sda2
fsck.ext4 /dev/sda2
reboot
resize2fs /dev/sda2
```
7. 查看ImmortalWrt上挂载的文件系统，这里已经变成1.97G了
![ImmortalWrt上挂载的文件系统](https://upload-images.jianshu.io/upload_images/12844614-b8350b1a22d80a26.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.6.安装主题和openclash
# 3.完成
