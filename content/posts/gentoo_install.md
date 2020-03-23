+++
title = "我的 Gentoo Linux 安装手册 2.0"
author = ["keke"]
date = 2020-02-16T00:00:00+08:00
draft = false
creator = "Emacs 26.3 (Org mode 9.3.6 + ox-hugo)"
+++

## last update:<span class="timestamp-wrapper"><span class="timestamp">&lt;2020-03-23 Mon&gt;</span></span> {#last-update}


## preview {#preview}

{{< figure src="https://s1.ax1x.com/2020/03/23/87jAeg.png" >}}


## 更新说明 {#更新说明}

2.0:

去除genfstab,mbr

内核由二进制改为gentoo-sources并由genkernel进行配置,ext4改为zfs,ubuntu livecd 改为 sysresccd,portage同步方式改为git

1.0:

将一直用的配置整理出来


## 使用sysresccd 作为 livecd {#使用sysresccd-作为-livecd}

<https://xyinn.org/gentoo/livecd/>


## 分区方案 {#分区方案}

| partition | type         | size   | mountpoint | formttype     |
|-----------|--------------|--------|------------|---------------|
| /dev/sdx1 | EFI System   | ~100Mb | /boot/efi  | mkfs.fat -F32 |
| /dev/sdx2 | Solaris root | full   | none       | none          |


## zfs 初始化 {#zfs-初始化}

```shell
#将/dev/sda2创建为zfs存储池
zpool create -f -o ashift=12 -o cachefile=/tmp/zpool.cache -O normalization=formD -O compression=lz4 -m none -R /mnt/gentoo kpool /dev/disk/by-id/ata-硬盘名字-2
#查看池状态
zpool status
#创建数据集
zfs create -o mountpoint=none -o canmount=off kpool/gentoo
zfs create -o mountpoint=/ kpool/gentoo/system
zfs create -o mountpoint=/boot kpool/gentoo/boot
zfs create -o mountpoint=/home kpool/home
#设置boot数据集可引导
zpool set bootfs=kpool/gentoo/boot kpool
#验证
zfs list -t all
zpool get bootfs rpool
```


## 下载并解压到根目录stage3文件 {#下载并解压到根目录stage3文件}

<https://mirrors.tuna.tsinghua.edu.cn/gentoo/>

进入镜像站的/gentoo/releases/amd64/autobuilds/目录，选择想要的stage3类型比如nomultilib(stage3-amd64-nomultilib-20200212T214502Z.tar.xz)

```sh
cd /mnt/gentoo
wget https://mirrors.tuna.tsinghua.edu.cn/gentoo/releases/amd64/autobuilds/current-stage3-amd64-nomultilib/stage3-amd64-nomultilib-20200212T214502Z.tar.xz
tar vxpf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```


## 配置make.conf&gentoo.conf,手动clone gentoo mirror repos {#配置make-dot-conf-and-gentoo-dot-conf-手动clone-gentoo-mirror-repos}

**my ryzen pc**

```conf
# GCC
CFLAGS="-march=znver1 -O2 -pipe"
CXXFLAGS="${CFLAGS}"
CHOST="x86_64-pc-linux-gnu"
CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt sse sse2 sse3 sse4_1 sse4_2 sse4a ssse3"
MAKEOPTS="-j13"

# USE
USE="-bindist pulseaudio"

# Portage
GENTOO_MIRRORS="https://mirrors.tuna.tsinghua.edu.cn/gentoo/"
EMERGE_DEFAULT_OPTS="--ask --verbose=y --keep-going --with-bdeps=y --load-average"
# FEATURES="${FEATURES} -userpriv -usersandbox -sandbox"
ACCEPT_KEYWORDS="~amd64"
ACCEPT_LICENSE="*"

# Language
L10N="en-US zh-CN en zh"
LINGUAS="en_US zh_CN en zh"

# Else
VIDEO_CARDS="amdgpu radeonsi"
GRUB_PLATFORMS="efi-64"
```

**my thinkpad x220 laptop**

```conf
# GCC
CFLAGS="-march=sandybridge -O2 -pipe"
CXXFLAGS="${CFLAGS}"
CHOST="x86_64-pc-linux-gnu"
CPU_FLAGS_X86="aes avx mmx mmxext popcnt sse sse2 sse3 sse4_1 sse4_2 ssse3"
MAKEOPTS="-j5"

# USE
USE="-bindist pulseaudio"

# Portage
GENTOO_MIRRORS="https://mirrors.tuna.tsinghua.edu.cn/gentoo/"
EMERGE_DEFAULT_OPTS="--ask --verbose=y --keep-going --with-bdeps=y --load-average"
# FEATURES="${FEATURES} -userpriv -usersandbox -sandbox"
ACCEPT_KEYWORDS="~amd64"
ACCEPT_LICENSE="*"

# Language
L10N="en-US zh-CN en zh"
LINGUAS="en_US zh_CN en zh"

# Else
VIDEO_CARDS="intel i965"
GRUB_PLATFORMS="efi-64"
```

```sh
mkdir /mnt/gentoo/etc/portage/repos.conf
nano /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```

```conf
[DEFAULT]
main-repo = gentoo

[gentoo]
location = /var/db/repos/gentoo
sync-type = git
sync-uri = https://github.com/gentoo-mirror/gentoo
auto-sync = yes
```

手动clone gentoo mirror repos

```sh
git clone https://github.com/gentoo-mirror/gentoo /mnt/gentoo/var/db/repos/gentoo
```


## 复制相关文件并chroot {#复制相关文件并chroot}

复制dns:

```sh
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```

挂载文件系统：

```sh
mount -t proc none proc
mount --rbind /sys sys
mount --rbind /dev dev
```

复制zpool.cache

```sh
mkdir -p /mnt/gentoo/etc/zfs
cp /tmp/zpool.cache /mnt/gentoo/etc/zfs/zpool.cache
```

进入chroot:

```sh
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
```

挂载efi分区：

```sh
mount /dev/sdx1 /boot/efi
```


## 选择Profile并更新系统 {#选择profile并更新系统}

安装基础程序:

```sh
emerge -av git eix gentoolkit
```

选择profile:

```sh
eselect profile list
eselect profile set x
```

更新系统:

```sh
eix-sync
emerge -avuDN @world
```


## 配置时区和地区 {#配置时区和地区}

```sh
echo "Asia/Shanghai" > /etc/timezone
emerge --config sys-libs/timezone-data

echo "en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8" >> /etc/locale.gen

locale-gen

eselect locale list

eselect locale set X

#修改主机名
echo hostname=\"Test\" > /etc/conf.d/hostname
```


## fstab {#fstab}

```sh
nano /etc/fstab
/dev/sda1               /boot/efi       vfat            noauto        1 2
```


## zfs 相关包的配置和use {#zfs-相关包的配置和use}

```sh
#zfs
echo "sys-fs/zfs-kmod **" >> /etc/portage/package.accept_keywords
echo "sys-fs/zfs **" >> /etc/portage/package.accept_keywords
#util-linux
echo ">=sys-apps/util-linux-2.30.2 static-libs" > /etc/portage/package.use/util-linux
#grub
echo "sys-boot/grub libzfs" > /etc/portage/package.use/grub
```


## 下载配置<内核,genkernel,zfs>,编译内核 {#下载配置-内核-genkernel-zfs-编译内核}

```sh
emerge -av gentoo-sources genkernel
```

先运行以下genkernel以得到一个.config文件，这个文件在安装ZFS的时候会检查

```sh
genkernel kernel --no-mountboot
```

安装zfs&grub

```sh
emerge -av zfs grub dhcpcd sudo
```

编译内核:

```sh
touch /boot/grub/grub.cfg
genkernel all --makeopts=-j4 --no-mountboot --zfs --bootloader=grub2 --callback="emerge @module-rebuild"
genkernel initramfs
```

安装引导:

```sh
grub-install --target=x86_64-efi --efi-directory=/boot/efi
grub-mkconfig -o /boot/grub/grub.cfg
```

启动zfs 服务:

```sh
rc-update add zfs-import boot
rc-update add zfs-mount boot
rc-update add zfs-share default
rc-update add zfs-zed default
```


## 修改root 密码，创建用户 {#修改root-密码-创建用户}

```sh
passwd
useradd -m -G users,wheel,portage,usb,video xx
passwd xx
```


## 准备结束 {#准备结束}

```sh
sed -i 's/\# \%wheel ALL=(ALL) ALL/\%wheel ALL=(ALL) ALL/g' /etc/sudoers
umount /dev/sdx1
exit
umount -lR {dev,proc,sys}
zpool export kpool
reboot
```