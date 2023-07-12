Title: 用 extroot 为 openwrt 扩充存储空间
Date: 2014-01-03 01:48
Category: IT 笔记
Tags: openwrt
Slug: openwrt-extroot
Authors: Kevin Chen

水星这款 MW4350r 内存为 128M，运行很多程序都不在话下。但是却只提供了 8M Flash 存储空间，而路由器系统还占了 1.9M，剩下的 5M 空间不足以支持安装很多软件，比如我在安装 python 的时候就报错提示存储空间不足，这确实很郁闷，但幸好 Openwrt 还提供了 extroot 方式来扩展存储，来发挥路由器和 Openwrt 系统的真正实力。

**pivot-overlay 还是 pivot-root？**

> 我把两种方式都试过，pivot-overlay 方式不能够把安装程序的位置移到 USB 存储装置上，但是 pivot-root 方式可以，所以我选择了后者。pivot-root 方式使/覆盖掉了/overlay 成为 rootfs，我认为这种方式更接近原生的 Linux 系统。  
> 而从官方的文档来看，目前 pivot-root 已经没有以前的缺点和不足，选择哪个已经是个人需求而不是技术问题了。  
> 网上大部分文章帖子都是 2009-2010 年间的，所以大部分可能都是 pivot-overlay 的。如果对这部分不太理解，请仔细阅读官方 Wiki：[ExtRoot: How it works][1],[The OpenWrt Flash Layout][2]

### 安装必要的包

```bash
opkg update
opkg install e2fsprogs kmod-usb-core kmod-usb2 kmod-usb-storage usbutils kmod-fs-ext4 block-mount
```

e2fsprogs 包提供了 mkfs（mkfs.ext3,mkfs.ext4）、fsck 等工具。  
kmod-usb2 只提供了 USb2.0 的驱动，如果你的是 USB1.0（1.1）的，还需要单独安装驱动。  
kmod-fs-ext4 是用来挂载 ext4 文件系统的，如果你想使用 ext3 文件格式就安装相应的包。  
usbutils 不是必装，仅提供了 lsusb 命令。

### 格式化 U 盘

插好 U 盘后，先查看下是否被系统识别出来

```bash
root@openwrt: ~# lsusb
Bus 001 Device 002: ID 0781:5571 SanDisk Corp. Cruzer Fit
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

```bash
root@openwrt: ~# ls /dev/sd*
/dev/sda    /dev/sda1
```

已经正确识别出来了，格式化系统为 ext4

```bash
mkfs.ext4 /dev/sda1
```

挂载到当前系统

```bash
mount /dev/sda1 /mnt
```

创建一个 128M 的 swap 文件

```bash
dd if=/dev/zero of=/mnt/swap bs=2048 count=65536
mkswap /mnt/swap
swapon /mnt/swap
```

### extroot

把/目录下的文件迁移到 U 盘,pivot-root 方式，适用于 Barrier Breaker（trunk）版本

```bash
mkdir -p /tmp/cproot
mount --bind / /tmp/cproot
tar -C /tmp/cproot -cvf - . | tar -C /mnt/ -xf -
umount /tmp/cproot
```

### 编辑 fstab

我的系统默认没有/etc/config/fstab 文件，可以用命令生成一个

```bash
block detect > /etc/config/fstab
```

编辑这个文件，添加下面这段配置

```bash
config mount
        option target        /
        option device        /dev/sda1
        option fstype        ext4
        option options       rw,sync
        option enabled       1
        option enabled_fsck  0
```

重启后，查看下空间，如果类似我这样就是成功了

```bash
root@openwrt: ~# df -h
Filesystem                Size      Used Available Use% Mounted on
rootfs                    7.2G    169.9M      6.7G   2% /
/dev/root                 1.8M      1.8M         0 100% /rom
tmpfs                    61.7M      1.0M     60.7M   2% /tmp
/dev/sda1                 7.2G    169.9M      6.7G   2% /
tmpfs                   512.0K         0    512.0K   0% /dev
```

大功告成，现在系统空间足够大了，任你怎么安装怎么下载。

[1]: http://wiki.openwrt.org/doc/howto/extroot/extroot.theory
[2]: http://wiki.openwrt.org/doc/techref/flash.layout
