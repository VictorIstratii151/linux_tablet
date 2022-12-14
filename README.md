# linux_tablet
Try to install linux on a chinese tablet
----------------------------------------
0. this algorithm works for both Inet D70 rev 3.0 with A23 processor and Q8 V3.0 with A33 processor
1. Download u-boot: git clone git://git.denx.de/u-boot.git
2. create u-boot-sunxi-with-spl.bin script:
defconfigs are located in u-boot/configs, choose one to do the fast compilation
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- <board_name>_defconfig
or configure the u-boot by yourself from console
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-
make
3. u-boot-sunxi-with-spl.bin should now be created in root directory of u-boot
4. Download linux kernel: git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git --depth=1
5. Configure the kernel: make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
6. Build the kernel image: ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j4 zImage
7. After compiling, should have a linux/arch/arm/boot/zImage
8. Compile the device tree: ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j4 <family>-<soc>-<board>.dtb (the .dts are located in linux/arch/arm/boot/dts)
9. A compiled device tree should be now located in linux/arch/arm/boot/dts/sun8i-a33-et-q8-v1.6.dtb
10. start preparing the sd card
11. export card=/dev/mmcblk0
12. export p=p
13. clean the card with: dd if=/dev/zero of=${card} bs=1M count=1
14. flash the bootloader to sd card with: dd if=u-boot-sunxi-with-spl.bin of=${card} bs=1024 seek=8   (note that it is not mounted nor partitioned yet)
15. partition the sd card:
blockdev --rereadpt ${card}
cat <<EOT | sfdisk ${card}
1M,16M,c
,,L
EOT
16. create the filesystem:
mkfs.vfat ${card}${p}1
mkfs.ext4 ${card}${p}2
cardroot=${card}${p}2
17. copy the files to first partition:
mount ${card}${p}1 /mnt/
cp boot.scr /mnt/    (will see it a bit later)
cp zImage /mnt/
cp <your_device>.dtb /mnt/ (for ex: cp sun8i-a33-et-q8-v1.6.dtb /mnt/)
umount /mnt/
18. Create a boot command that can be either:
setenv bootargs console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait panic=10
load mmc 0:1 0x43000000 ${fdtfile} || load mmc 0:1 0x43000000 boot/${fdtfile}
load mmc 0:1 0x42000000 zImage || load mmc 0:1 0x42000000 boot/zImage
bootz 0x42000000 - 0x43000000

(instead of fdtfile just better put in the name of .dtb used in your setup)

or

load mmc 0:1 0x43000000 ${fdtfile} || load mmc 0:1 0x43000000 boot/${fdtfile}
load mmc 0:1 0x42000000 zImage || load mmc 0:1 0x42000000 boot/zImage
bootz 0x42000000 - 0x43000000

if you passed the boot args to u-boot

19. create the boot script from this command: mkimage -C none -A arm -T script -d boot.cmd boot.scr
20. Download a rootfs (for example debian): https://uk.lxd.images.canonical.com/images/debian/stretch/armhf/default/20220128_05:25/
21. mount the rootfs to the second partition:
mount ${card}${p}2 /mnt/
tar -C /mnt/ -xjpf rootfs.tar.xz
umount /mnt
22. try not to die
23. USB PROBLEM SOLUTION!!!!
KERNEL IMAGE!
using mainline kernel (6.1), create zImage using sunxi_defconfig. As of 12.2022,
sunxi_defconfig made the USB magically work after kernel boot, but it made the display
just show a blank black screen with no console. After comparing the .config made using sunxi_defconfig (where usb works) with .config created after lurking through menuconfig, I have found that sunxi_defconfig doesn't set the CONFIG_FB=y, which allows modifying the framebuffer console. I had to manually set this value in .config, then launch menuconfig and run LOAD in order to load the configuration. Afterwards, it was possible to find Console/Framebuffer support (in Graphics Support submenu), set CONFIG_FRAMEBUFFER_CONSOLE,  CONFIG_FRAMEBUFFER_CONSOLE_DETECT_PRIMARY, CONFIG_FRAMEBUFFER_CONSOLE_ROTATION in console support, as well as CONFIG_FB_SIMPLE in frame buffer support submenu. If I didn't miss anything, this makes the display work properly for sunxi_defconfig.
Good to have .config support enabled in menuconfig to allow reverse engineering the configs form zImage
Also, because wifi drivers were compiled as modules, RFKILL had to be enabled too in menuconfig, so that the wireless adapter could be started properly.


DTB!
I've found out that no matter how hard I tried, the .dtb created using sun8i-a33-q8-tablet.dts from LINUX repo,
the USB was just DEAD after kernel boot, even with zImage configured as above. The Internet suggested me to use the same .dtb created within U-BOOT repo as a result of compiling the bootloader. THE ONLY THING I HAD TO CHANGE IN DTS was dr_mode=host for &usb_otg in sun8i-reference-design-tablet.dtsi, so that the unified .dts had it properly set with status=okay. Setting this value anywhere else results in duplicated status field, which probably breaks the resulting blob.

ROOTFS!
So far I used https://olimex.wordpress.com/2014/07/21/how-to-create-bare-minimum-debian-wheezy-rootfs-from-scratch/ in order to set up a minimal rootfs with no UI. In addition I installed different tools within chroot, like net-tools, network-managers (maybe it's not needed?). For UI (will work on this later) I had to install java suitable for the Debian distro of choice, then use tasksel (or something like this?) to download the UI packages. Not sure if editing /etc/network/interfaces made anything useful for the wifi configs (about wifi below)

Don't forget to edit/create fstab!
none		/tmp	tmpfs	defaults,noatime,mode=1777	0	0 # <-- tmpfs
/dev/mmcblk0p2	/	ext4	defaults	0	1 
/dev/mmcblk0p1	/boot	vfat	defaults	0	2 
https://linux-sunxi.org/Debootstrap

24. compiling/decompiling a device tree
dtc -I dtb -O dts sun8i-a33-q8-tablet.dtb -o sun8i-a33-q8-tablet.dts
dtc -I dts -O dtb -f sun8i-a33-q8-tablet.dts -o sun8i-a33-q8-tablet.dtb

25. installing wifi:
Unfortunately both devices have wifi chips that are not supported by kernel menuconfig out of the box.
*A23 device wifi chipset - Realtek RTL8189ETV*
Found the information here: https://linux-sunxi.org/Wifi#RTL8189ES_.2F_RTL8189ETV
git clone https://github.com/jwrdegoede/rtl8189ES_linux.git
cd rtl8189ES_linux.git
make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnu- KSRC=../linux

here ../linux is the linux repo where I have compiled my working zImage prior to building the module

*A33 device wifi chipset - Realtek RTL8703BS*
This case turned out to be a lot tricker than for A23. There is practically no information about rtl8703bs and it's drivers for linux, but I've found that it is pin-to-pin compatible with another chip: *RTL8723CS*. This link https://linux-sunxi.org/Finepower_N1 told me that I can also download the repo with the driver and build it. Unfortunately, the repo https://github.com/Icenowy/rtl8723cs/tree/new-driver-by-megous was not maintained for a long time, and trying to build the driver form it resulted into nothing. After personally talking to Icenowy, I learned that I can pull this repo: https://github.com/megous/linux/tree/wifi-6.1 (checked out the branch wifi-6.1 which had the drivers updated for my specific version of mainline kernel) and build the driver externally by using the same working linux repo that I had. It didn't go so well out of the box, I had to change the makefile and a couple of headers (see my repo with this driver). 

make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- KSRC=../linux M=$PWD   <-- from within the driver directory.


After building the driver module, the whole folder has to be moved to p2 of the sd card. I created a folder /usr/lib/kernel/*kernelver*/ where each driver folder could be put. 
After doing this, launch the tablet, go inside the necessary driver directory and run insmod <driver>.ko 
(e.g. insmod 8189es.ko). 
Can invoke lsmod (probably have to install kmod to use this command) 
Then, can also invoke lshw in order to see what hardware is detected by the system. If the wireless is displayed, then you're good to go.
Steps to configure, start and connect to wifi are described here: https://linuxcommando.blogspot.com/2013/10/how-to-connect-to-wpawpa2-wifi-network.html
