# Odroid-XU-Xen-setup
* Reference by 
```
http://wiki.xenproject.org/wiki/Xen_ARM_with_Virtualization_Extensions/OdroidXU#Booting_from_SD.2FeMMC_card
http://forum.odroid.com/viewtopic.php?f=64&t=3831
http://odroid.com/dokuwiki/doku.php?id=en:xu4_xen
http://www.armhf.com/boards/odroid-xu/odroid-sd-install/
```
* Preparing  
Prepare a microSD Card.  
Use minicom: `sudo minicom -c on`  
SD card device name assumption: `/dev/sdc`  
SD card mount point assumption: `~/boot, ~/rootfs`  
linux dirctory assumption: `~/linux-xen`  
xen dirctory assumption: `~/xen`  
`apt-get install gcc-4.9-arm-linux-gnueabihf`   
# (gcc version 4.9.3 (Ubuntu/Linaro 4.9.3-4ubuntu1) )  
# gcc version over 4.9 will not work.  
`ln -s /usr/bin/arm-linux-gnueabihf-gcc-4.9 /usr/bin/arm-linux-gnueabihf-gcc`

##ODROID-XU microSD Card Installation ##
* Partition
`fdisk /dev/sdc`
```
o p
n p 1 enter +16M
t e
n p 2 enter enter 
w
```
Format partition 1 as FAT by typing `mkfs.vfat /dev/sdc1`  
Format partition 2 as ext4 by typing `mkfs.ext4 /dev/sdc2`

* U-boot
`git clone -b odroid-3.13.y https://github.com/suriyanr/linux-xen.git --depth=1`  
`~/linux-xen/sd_fuse/sd_fusing.sh /dev/sdc`

* Building a Linux Dom0 Rootfs 
```
wget http://s3.armhf.com/dist/odroid/ubuntu-trusty-14.04-rootfs-3.4.91.1-odroidxu-armhf.com.tar.xz
mkdir boot
mkdir rootfs
mount /dev/sdc1 boot
mount /dev/sdc2 rootfs
tar xJvf ubuntu-trusty-14.04-rootfs-3.4.91.1-odroidxu-armhf.com.tar.xz -C rootfs
```
dom0 login userid is `ubuntu` and the password is `ubuntu`. 

*  Building XEN 
```
git clone git://xenbits.xen.org/xen.git -b staging --depth 1 xen-staging
cd xen
export CROSS_COMPILE=arm-linux-gnueabihf-	
make dist-xen XEN_TARGET_ARCH=arm32 debug=y CONFIG_EARLY_PRINTK=exynos5250
mkdir ~/boot/xen
mkimage -A arm -T kernel -a 0x80200000 -e 0x80200000 -C none -d "$~/boot/xen" xen4.5-uImage
cd ~
```

* Linux Dom0 kernel and modules 
```
cd linux-xen
export CROSS_COMPILE=arm-linux-gnueabihf-
make odroidxu_xen_defconfig
make ARCH=arm CROSS_COMPILE=$CROSS_COMPILE odroidxu_xen_defconfig
make ARCH=arm CROSS_COMPILE=$CROSS_COMPILE zImage
cp arch/arm/boot/zImage ~/boot/xen
make ARCH=arm CROSS_COMPILE=$CROSS_COMPILE dtbs
cp arch/arm/boot/dts/exynos5410-odroidxu.dtb ~/boot/xen
make ARCH=arm CROSS_COMPILE=$CROSS_COMPILE modules
make ARCH=arm CROSS_COMPILE=$CROSS_COMPILE INSTALL_MOD_PATH=~/rootfs modules_install
cp ~/linux-arm/sd_fuse/boot.ini ~/boot/
cd ~
```
* Console login prompt 
`cp ~/rootfs/etc/init/tty1.conf ~/rootfs/etc/init/hvc0.conf`  
motify ~/rootfs/etc/init/console.conf 
```
start on stopped rc RUNLEVEL=[2345]
stop on runlevel [!2345]
respawn
exec /sbin/getty -8 38400 console
```
*  Building a Linux DomU Kernel 
```
cd linux-xen
make ARCH=arm CROSS_COMPILE=$CROSS_COMPILE exynos_defconfig
make ARCH=arm menuconfig
```
When presented with the menu, make sure the below are enabled (press y): 
```
 1. Kernel Features -> Xen guest support on ARM
 2. Device Drivers -> Block devices -> Xen virtual block device support.
 3. Device Drivers -> Network device support -> Xen network device frontend
 4. Device Drivers -> Xen driver support -> Select all.
 5. System Type -> ARM system type -> Allow multiple platforms to be selected.
 6. System Type -> Multiple platform selection -> ARMv7 based platforms
 7. System Type -> Dummy Virtual Machine.
 8. Device Drivers -> Input Device support -> Miscellaneous devices -> Xen virtual keyboard and mouse support.
```
 * Disabled (press n): 
```
 9. Networking support -> Networking options -> TCP/IP networking -> The IPv6 protocol 
```
```
cp -a arch/arm/mach-exynos/include/ arch/arm/include/ (Because the include not found)
make ARCH=arm CROSS_COMPILE=$CROSS_COMPILE zImage -j4
cp arch/arm/boot/zImage ~/rootfs/root/
cd ~
```
*  Building a Linux DomU Rootfs 
```
wget http://odroid.in/ubuntu_14.04lts/ubuntu-14.04lts-xubuntu-odroid-xu-20140714.img.xz
unxz ubuntu-14.04lts-xubuntu-odroid-xu-20140714.img.xz
kpartx -v -a ubuntu-14.04lts-xubuntu-odroid-xu-20140714.img.xz
dd if=/dev/mapper/loop0p2 of=domU.img
mkdir domU-root
mount domU.img domU-root
cd domU-root
```
`vi ./etc/fstab`  
./etc/fstab
```
/dev/xvda    /    ext4    errors=remount-ro    0    1
```
`vi ./etc/init/console.conf`  
./etc/init/console.conf  
```
start on stopped rc RUNLEVEL=[2345]
stop on runlevel [!2345]

respawn
exec /sbin/getty -8 38400 hvc0
```
```
cd ~
umount domU-root
cp domU.img ~/rootfs/root/
sync
cd ~/rootfs/root
```
`vi domU.cfg`  
domU.cfg
```
kernel = "/root/Image"
name = "guest"
memory = 512
vcpus = 1
extra = "console=hvc0 earlyprintk debug root=/dev/xvda ro"
disk = [ 'phy:/root/domU.img,xvda,w' ]
vif = [ '' ]
```
* Running virtual machine dom0 :  
`umount /dev/sdc1`   
`umount /dev/sdc2`   
eject SDcard from PC and insert SDcard to odroid-xu and then boot it.  
dom0 login userid is `ubuntu` and the password is `ubuntu`.   

* Domain 0 Settings  
xen tools:
```
$ git clone  https://github.com/bkrepo/xen.git
$ cd xen
$ apt-get update
$ apt-get build-dep xen
$ apt-get install libpixman-1-dev
$ ./configure --disable-xen --disable-docs
$ make dist-tools -j4
$ make install-tools
$ update-rc.d xencommons defaults 19 18
$ update-rc.d xendomains defaults 21 20
$ update-rc.d xen-watchdog defaults 22 23
$ ldconfig
```
Network configuration:
```
$ apt-get install bridge-utils
```
`vim /etc/network/interfaces.d/xenbr0.cfg`
```
auto xenbr0 inet
iface xenbr0 inet dhcp
bridge_ports eth0
```

* Running virtual machine domU :  
`xl create -c /root/xen.cfg`  
id: 'odroid', passwd: 'odroid'   

* There are some problem :  
domU doesn't have network now.

