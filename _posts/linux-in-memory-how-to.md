---
title: 在 Linux 启动后去掉启动介质
date: 2016-04-25 11:49:51
tags: 
- 技术
- 瞎搞
- Linux
---

## 缘起

最近总是会有一些需求，就是快速地部署一个临时的网关。有的时候，用于部署网关的电脑可能只是临时拿来用的。因此，给人家重新装个系统就很不靠谱了。我通常的做法是，在我的 U 盘里装个 ubuntu 之类的。然后里边装上 bind、isc-dhcp-server 之类的，然后每次用这个 U 盘启动就好了。后来发现，这个问题并不是那么简单，因为 U 盘插在别人的电脑上，一来容易丢失，二来容易被物理碰撞损坏。于是我就考虑，能不能在启动后，把根文件系统载入到内存中，这样就可以拔掉 U 盘了。

<!--more-->

## 一种方案

有一种方案是显然可行的，就是把整个系统搞成一个 initrd，这样自然就在内存中了。这样作的弊端是，initrd 是 bootloader 载入到内存中的。而 Grub 读取硬盘的驱动是走的 BIOS，这样速度就很慢了。同时，尽可能少的改动发行版，也有利于后续继续安装软件和维护。

## 我的思路

我的思路是，在 initrd 执行完毕后，替换掉原系统的 init 程序，换上去我的，然后 mount 上去一个 tmpfs，然后把根文件系统拷贝过去，最后 chroot 进去，起里边原来的 init。

虽说思路是很简单的，但是要想实现起来，还是有一些细节要考虑的，主要要点是：

- 合理地把之前挂上去的 / 给 umount 掉

为此，则必须用一些神奇的操作来解除对原来的 root 的占用。

## 实现方法

写一个脚本，放在 `/usr/local/sbin/init.sh` 下，内容是：

``` bash
#!/bin/bash

read -p "Input 'y' in 5 seconds to boot normally..." -t 5 yes

if [ \( x$yes = xy \) -o \( x$yes = xY \) ]; then
  exec /sbin/init "$@"
  echo failed...
  sleep 10000
fi

echo Reading rootfs, it may take several minutes...
mkdir -p /run/rootfs
mount -t tmpfs  -o size=4G shankers-mem-ubuntu /run/rootfs 
rsync -a / /run/rootfs/ --exclude=/proc --exclude=/dev --exclude=/sys --exclude=/run --exclude=/var/cache --exclude=/var/log --exclude=/usr/include --exclude=/usr/local/include 
cd /run/rootfs
for i in proc dev sys run var/cache var/cache/bind var/log; do
  mkdir -p $i
done

mount -t proc mem_proc proc
mount -t sysfs mem_sys sys
mount -t tmpfs mem_run run
mount -t devtmpfs mem_dev dev
mount -t devpts  mem_devpts dev/pts
mount -t tmpfs mem_tmpfs tmp

echo > etc/fstab
mkdir oldroot
#exec /bin/bash
pivot_root . oldroot
###
# 这里之所以 >dev/console，是因为现在的 init.sh 的 
# stdin、stdout 和 stderr 原本指向了 /dev/console
# 由于 /dev 是挂载在原来的 root 下的，pivot_root 后
# 跑到了 /oldroot/dev/console 中。如果不加上 >dev/console，
# 就会保持 /oldroot/dev/console 打开，导致 /oldroot 
# umount 不下来。
###
exec chroot . bin/bash -s "$@" >dev/console 2>&1 << 'HERE'
cd /
umount -R oldroot
rmdir oldroot
echo Will start mem system in 5 seconds...
sleep 5
exec /sbin/init "$@" </dev/console >/dev/console 2>&1
echo failed...
sleep 10000
HERE
```
别忘了 `chmod +x /usr/local/sbin/init.sh`。

之后新建一个 `/etc/default/grub.d/memroot.cfg`，里边写上：

``` bash
GRUB_CMDLINE_LINUX_DEFAULT="$GRUB_CMDLINE_LINUX_DEFAULT init=/usr/local/sbin/init.sh"
```

最后 

```
update-grub
```

就可以了。

以上脚本在 Ubuntu 16.04 下测试通过。
