# linux_tablet
Try to install linux on a chinese tablet
----------------------------------------

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

or

load mmc 0:1 0x43000000 ${fdtfile} || load mmc 0:1 0x43000000 boot/${fdtfile}
load mmc 0:1 0x42000000 zImage || load mmc 0:1 0x42000000 boot/zImage
bootz 0x42000000 - 0x43000000

if you passed the boot args to u-boot

19. create the boo script from this command: mkimage -C none -A arm -T script -d boot.cmd boot.scr
20. Download a rootfs (for example debian): https://uk.lxd.images.canonical.com/images/debian/stretch/armhf/default/20220128_05:25/
21. mount the rootfs to the second partition:
mount ${card}${p}2 /mnt/
tar -C /mnt/ -xjpf rootfs.tar.xz
umount /mnt
22. try not to die

