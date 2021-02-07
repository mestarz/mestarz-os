### 1 安装qemu
```shell
sudo apt install qemu
sudo apt install qemu-system-x86_64
```
### 2 制作磁盘镜像
#### 2.1 创建磁盘
使用 `qemu-img` 创建一个512M的磁盘镜像文件      
```shell
qemu-img create -f raw disk.raw 512M
```
使用 `mkfs` 对磁盘镜像进行格式化
```shell
mkfs -t ext4 ./disk.raw
```
#### 2.2 挂载磁盘镜像
使用 `mount` 以loop方式将磁盘文件挂载到一个目录上
```shell
sudo mount -o loop ./disk.raw ./img
```
#### 2.3 安装内核模块
在linux源码目录下运行
```shell
sudo make modules_install \  #安装内核模块
INSTALL_MOD_PATH=./img		   #指定安装路径
```
#### 2.4 配置init程序
参考busybox[代码文档](https://git.busybox.net/busybox/tree/examples/inittab)可知，init启动后会扫描 `/etc/inittab` 配置文件，这个配置文件决定了init程序的行为。而busybox init在没有 `/etc/inittab` 文件的情况下也能工作，因为它有默认行为。它的默认行为相当于如下配置：
```shell
::sysinit:/etc/init.d/rcS
::askfirst:/bin/sh
::ctrlaltdel:/sbin/reboot
::shutdown:/sbin/swapoff -a
::shutdown:/bin/umount -a -r
::restart:/sbin/init
tty2::askfirst:/bin/sh
tty3::askfirst:/bin/sh
tty4::askfirst:/bin/sh
```
参考文档，进行如下操作
```shell
cd ./img
sudo mkdir ./etc
cd ./etc
sudo vim inittab
```
输入配置如下：
```shell
::sysinit:/etc/init.d/rcS
console::askfirst:-/bin/sh
::ctrlaltdel:/sbin/reboot
::shutdown:/sbin/swapoff -a
::shutdown:/bin/umount -a -r
::restart:/sbin/init
```
创建可执行文件 `/etc/init.d/rcS` 
```shell
cd ./img
sudo mkdir ./etc/init.d
cd ./etc/init.d
sudo vim rcS
sudo chmod 777 rcS
```
程序内容如下（空程序）：
```shell
#!/bin/sh
```
### 3 制作init程序
#### 3.1 编译busybox程序
下载busybox源码，进入源码目录，启用默认配置
```shell
make defconfig
```
使用菜单程序定制配置
```shell
make menuconfig
```
因为磁盘镜像中没有程序库，需要将busybox编译成一个独立、无依赖的可执行程序，窗口配置路径如下：
```shell
Settings --->
	--- Build Options
  [*] Build static binary (no shared libs)
```
配置完后执行编译
```shell
make
```
#### 3.2 安装busybox到磁盘镜像
```shell
make CONFIG_PREFIX=./img \  #磁盘文件的挂载路径
	install	
```
### 4 qemu启动内核
```shell
qemu-system-x86_64 \
    -m 512M \																#指定内存大小
    -smp 2 \    														#指定虚拟的CPU数量
    -kernel ./bzImage \											#指定内核文件路径
    -drive format=raw,file=./disk.raw \			#指定文件为磁盘
    -append "init=/linuxrc root=/dev/sda"		#内核启动参数
```
### 5 挂载文件系统
创建/dev，/proc，/sys目录
```shell
cd ./img
sudo mkdir ./dev
sudo mkdir ./proc
sudo mkdir ./sys
```
其中，/dev目录会在重启系统时自动挂载，/proc，/sys需要手动挂载，通过修改 `./etc/init.d/rcS` 文件使计算机自动挂载，修改 `./etc/init.d/rcS` 内容如下：
```shell
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
```





