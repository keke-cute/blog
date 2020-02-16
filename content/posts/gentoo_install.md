+++
title = "以 Arch Linux 的方式使用 Gentoo Linux"
author = ["keke"]
date = 2020-02-16T00:00:00+08:00
draft = false
creator = "Emacs 26.3 (Org mode 9.3.6 + ox-hugo)"
+++

## 分区并挂载 {#分区并挂载}

| boot type   | partition | type  | size      | mount | formt         |
|-------------|-----------|-------|-----------|-------|---------------|
| uefi(gpt)   | nvme0n1p1 | efi   | <100mb    | /boot | mkfs.fat -F32 |
|             | nvme0n1p2 | linux | full size | /     | mkfs.ext4     |
| legacy(dos) | nvme0n1p1 | linux | full size | /     | mkfs.ext4     |

创建目录并挂载：

```sh
mkdir -v /mnt/gentoo
mount -v /dev/nvme0n1px /mnt/gentoo
```


## 下载并解压到根目录stage3文件 {#下载并解压到根目录stage3文件}

<https://mirrors.tuna.tsinghua.edu.cn/gentoo/>

进入镜像站的/gentoo/releases/amd64/autobuilds/目录，选择想要的stage3类型比如nomultilib(stage3-amd64-nomultilib-20200212T214502Z.tar.xz)

```sh
cd /mnt/gentoo
wget https://mirrors.tuna.tsinghua.edu.cn/gentoo/releases/amd64/autobuilds/current-stage3-amd64-nomultilib/stage3-amd64-nomultilib-20200212T214502Z.tar.xz
tar vxpf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```


## 配置make.conf&gentoo.conf {#配置make-dot-conf-and-gentoo-dot-conf}

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
```

```sh
mkdir /mnt/gentoo/etc/portage/repos.conf
nano /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```

```conf
[gentoo]
location = /usr/portage
sync-type = rsync
sync-uri = rsync://mirrors.tuna.tsinghua.edu.cn/gentoo-portage/
#sync-uri = rsync://rsync.mirrors.ustc.edu.cn/gentoo-portage/
auto-sync = yes
```


## chroot {#chroot}

复制dns:

```sh
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```

挂载文件系统：

```sh
mount -t proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
```

如果我是用的ubuntu livecd 就需要：

```sh
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm
chmod 1777 /dev/shm
```

进入chroot:

```sh
chroot /mnt/gentoo /bin/bash
source /etc/profile
```

如果是uefi 启动就需要挂载boot 分区：

```sh
mount /dev/nvme0n1p1 /boot
```


## 选择Profile并更新系统 {#选择profile并更新系统}

使用快照更新Portage:

```sh
emerge-webrsync
```

使用rsync同步:

```sh
emerge --sync
```

选择prifile 并更新系统:

```sh
eselect profile list
eselect profile set x
emerge -auvDN --with-bdeps=y @world
```

如果碰到未满足的xxx或者其它提示:

```sh
emerge -auvDN --with-bdeps=y --autounmask-write @world
etc-update # 然后输入-3就能更新配置,确保再次运行时没有可更新的文件
emerge -auvDN --with-bdeps=y @world
```

更新完毕后确定没有更新了：

```sh
emerge @preserved-rebuild
perl-cleaner --all
emerge -auvDN --with-bdeps=y @world
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
```


## 像 Arch Linux 一样 genfstab {#像-arch-linux-一样-genfstab}

```sh
wget https://raw.githubusercontent.com/YangMame/Gentoo-Installer/master/genfstab
chmod +x genfstab
#可选 cp genfstab /usr/bin/
./genfstab / > /etc/fstab
nano /etc/fstab #最好检查下此文件,删掉无用挂载点
```


## NetworkManager {#networkmanager}

```sh
emerge -av networkmanager
#如果它说有未满足的xxxx或者其它提示:
emerge --autounmask-write networkmanager
etc-update --automode -3
emerge networkmanager
#openRC添加开机服务:
rc-update add NetworkManager default
#修改主机名
echo hostname=\"Test\" > /etc/conf.d/hostname
```


## 基础工具 {#基础工具}

```sh
emerge app-admin/sysklogd sys-process/cronie sudo grub
sed -i 's/\# \%wheel ALL=(ALL) ALL/\%wheel ALL=(ALL) ALL/g' /etc/sudoers
passwd
rc-update add sysklogd default
rc-update add cronie default
```


## 像 Arch Linux 一样使用二进制内核 {#像-arch-linux-一样使用二进制内核}

```sh
emerge --ak gentoo-kernel-bin
#uefi
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Gentoo
grub-mkconfig -o /boot/grub/grub.cfg
#legacy
grub-install --target=i386-pc /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```


## 创建用户 {#创建用户}

```sh
useradd -m -G users,wheel,portage,usb,video xx
passwd xx
```


## 显卡及Xorg {#显卡及xorg}

```sh
emerge --ask x11-drivers/xf86-video-intel(amdgpu)
#不要忘了firmware
emerge --ask linux-firmware
```

结束！


## 附录 {#附录}

如果网络不通可以直接复制genfatab 代码：

```sh
#!/bin/bash

shopt -s extglob

# generated from util-linux source: libmount/src/utils.c
declare -A pseudofs_types=([anon_inodefs]=1
			   [autofs]=1
			   [bdev]=1
			   [binfmt_misc]=1
			   [cgroup]=1
			   [configfs]=1
			   [cpuset]=1
			   [debugfs]=1
			   [devfs]=1
			   [devpts]=1
			   [devtmpfs]=1
			   [dlmfs]=1
			   [fuse.gvfs-fuse-daemon]=1
			   [fusectl]=1
			   [hugetlbfs]=1
			   [mqueue]=1
			   [nfsd]=1
			   [none]=1
			   [pipefs]=1
			   [proc]=1
			   [pstore]=1
			   [ramfs]=1
			   [rootfs]=1
			   [rpc_pipefs]=1
			   [securityfs]=1
			   [sockfs]=1
			   [spufs]=1
			   [sysfs]=1
			   [tmpfs]=1)

# generated from: pkgfile -vbr '/fsck\..+' | awk -F. '{ print $NF }' | sort
declare -A fsck_types=([cramfs]=1
		       [exfat]=1
		       [ext2]=1
		       [ext3]=1
		       [ext4]=1
		       [ext4dev]=1
		       [jfs]=1
		       [minix]=1
		       [msdos]=1
		       [reiserfs]=1
		       [vfat]=1
		       [xfs]=1)

out() { printf "$1 $2\n" "${@:3}"; }
error() { out "==> ERROR:" "$@"; } >&2
msg() { out "==>" "$@"; }
msg2() { out "  ->" "$@";}
die() { error "$@"; exit 1; }

ignore_error() {
  "$@" 2>/dev/null
  return 0
}

in_array() {
  local i
  for i in "${@:2}"; do
    [[ $1 = "$i" ]] && return 0
  done
  return 1
}

chroot_add_mount() {
  mount "$@" && CHROOT_ACTIVE_MOUNTS=("$2" "${CHROOT_ACTIVE_MOUNTS[@]}")
}

chroot_maybe_add_mount() {
  local cond=$1; shift
  if eval "$cond"; then
    chroot_add_mount "$@"
  fi
}

chroot_setup() {
  CHROOT_ACTIVE_MOUNTS=()
  [[ $(trap -p EXIT) ]] && die '(BUG): attempting to overwrite existing EXIT trap'
  trap 'chroot_teardown' EXIT

  chroot_maybe_add_mount "! mountpoint -q '$1'" "$1" "$1" --bind &&
  chroot_add_mount proc "$1/proc" -t proc -o nosuid,noexec,nodev &&
  chroot_add_mount sys "$1/sys" -t sysfs -o nosuid,noexec,nodev,ro &&
  ignore_error chroot_maybe_add_mount "[[ -d '$1/sys/firmware/efi/efivars' ]]" \
      efivarfs "$1/sys/firmware/efi/efivars" -t efivarfs -o nosuid,noexec,nodev &&
  chroot_add_mount udev "$1/dev" -t devtmpfs -o mode=0755,nosuid &&
  chroot_add_mount devpts "$1/dev/pts" -t devpts -o mode=0620,gid=5,nosuid,noexec &&
  chroot_add_mount shm "$1/dev/shm" -t tmpfs -o mode=1777,nosuid,nodev &&
  chroot_add_mount run "$1/run" -t tmpfs -o nosuid,nodev,mode=0755 &&
  chroot_add_mount tmp "$1/tmp" -t tmpfs -o mode=1777,strictatime,nodev,nosuid
}

chroot_teardown() {
  umount "${CHROOT_ACTIVE_MOUNTS[@]}"
  unset CHROOT_ACTIVE_MOUNTS
}

try_cast() (
  _=$(( $1#$2 ))
) 2>/dev/null

valid_number_of_base() {
  local base=$1 len=${#2} i=

  for (( i = 0; i < len; i++ )); do
    try_cast "$base" "${2:i:1}" || return 1
  done

  return 0
}

mangle() {
  local i= chr= out=

  unset {a..f} {A..F}

  for (( i = 0; i < ${#1}; i++ )); do
    chr=${1:i:1}
    case $chr in
      [[:space:]\\])
	printf -v chr '%03o' "'$chr"
	out+=\\
	;;
    esac
    out+=$chr
  done

  printf '%s' "$out"
}

unmangle() {
  local i= chr= out= len=$(( ${#1} - 4 ))

  unset {a..f} {A..F}

  for (( i = 0; i < len; i++ )); do
    chr=${1:i:1}
    case $chr in
      \\)
	if valid_number_of_base 8 "${1:i+1:3}" ||
	    valid_number_of_base 16 "${1:i+1:3}"; then
	  printf -v chr '%b' "${1:i:4}"
	  (( i += 3 ))
	fi
	;;
    esac
    out+=$chr
  done

  printf '%s' "$out${1:i}"
}

optstring_match_option() {
  local candidate pat patterns

  IFS=, read -ra patterns <<<"$1"
  for pat in "${patterns[@]}"; do
    if [[ $pat = *=* ]]; then
      # "key=val" will only ever match "key=val"
      candidate=$2
    else
      # "key" will match "key", but also "key=anyval"
      candidate=${2%%=*}
    fi

    [[ $pat = "$candidate" ]] && return 0
  done

  return 1
}

optstring_remove_option() {
  local o options_ remove=$2 IFS=,

  read -ra options_ <<<"${!1}"

  for o in "${!options_[@]}"; do
    optstring_match_option "$remove" "${options_[o]}" && unset 'options_[o]'
  done

  declare -g "$1=${options_[*]}"
}

optstring_normalize() {
  local o options_ norm IFS=,

  read -ra options_ <<<"${!1}"

  # remove empty fields
  for o in "${options_[@]}"; do
    [[ $o ]] && norm+=("$o")
  done

  # avoid empty strings, reset to "defaults"
  declare -g "$1=${norm[*]:-defaults}"
}

optstring_append_option() {
  if ! optstring_has_option "$1" "$2"; then
    declare -g "$1=${!1},$2"
  fi

  optstring_normalize "$1"
}

optstring_prepend_option() {
  local options_=$1

  if ! optstring_has_option "$1" "$2"; then
    declare -g "$1=$2,${!1}"
  fi

  optstring_normalize "$1"
}

optstring_get_option() {
  local opts o

  IFS=, read -ra opts <<<"${!1}"
  for o in "${opts[@]}"; do
    if optstring_match_option "$2" "$o"; then
      declare -g "$o"
      return 0
    fi
  done

  return 1
}

optstring_has_option() {
  local "${2%%=*}"

  optstring_get_option "$1" "$2"
}

dm_name_for_devnode() {
  read dm_name <"/sys/class/block/${1#/dev/}/dm/name"
  if [[ $dm_name ]]; then
    printf '/dev/mapper/%s' "$dm_name"
  else
    # don't leave the caller hanging, just print the original name
    # along with the failure.
    print '%s' "$1"
    error 'Failed to resolve device mapper name for: %s' "$1"
  fi
}

fstype_is_pseudofs() {
  (( pseudofs_types["$1"] ))
}

fstype_has_fsck() {
  (( fsck_types["$1"] ))
}


write_source() {
  local src=$1 spec= label= uuid= comment=()

  label=$(lsblk -rno LABEL "$1" 2>/dev/null)
  uuid=$(lsblk -rno UUID "$1" 2>/dev/null)

  # bind mounts do not have a UUID!

  case $bytag in
    '')
      [[ $uuid ]] && comment=("UUID=$uuid")
      [[ $label ]] && comment+=("LABEL=$(mangle "$label")")
      ;;
    LABEL)
      spec=$label
      [[ $uuid ]] && comment=("$src" "UUID=$uuid")
      ;;
    UUID)
      spec=$uuid
      comment=("$src")
      [[ $label ]] && comment+=("LABEL=$(mangle "$label")")
      ;;
    *)
      [[ $uuid ]] && comment=("$1" "UUID=$uuid")
      [[ $label ]] && comment+=("LABEL=$(mangle "$label")")
      [[ $bytag ]] && spec=$(lsblk -rno "$bytag" "$1" 2>/dev/null)
      ;;
  esac

  [[ $comment ]] && printf '# %s\n' "${comment[*]}"

  if [[ $spec ]]; then
    printf '%-20s' "$bytag=$(mangle "$spec")"
  else
    printf '%-20s' "$(mangle "$src")"
  fi
}

optstring_apply_quirks() {
  local varname=$1 fstype=$2

  # SELinux displays a 'seclabel' option in /proc/self/mountinfo. We can't know
  # if the system we're generating the fstab for has any support for SELinux (as
  # one might install Arch from a Fedora environment), so let's remove it.
  optstring_remove_option "$varname" seclabel

  case $fstype in
    f2fs)
      # These are Kconfig options for f2fs. Kernels supporting the options will
      # only provide the negative versions of these (e.g. noacl), and vice versa
      # for kernels without support.
      optstring_remove_option "$varname" noacl,acl,nouser_xattr,user_xattr
      ;;
    vfat)
      # Before Linux v3.8, "cp" is prepended to the value of the codepage.
      if optstring_get_option "$varname" codepage && [[ $codepage = cp* ]]; then
	optstring_remove_option "$varname" codepage
	optstring_append_option "$varname" "codepage=${codepage#cp}"
      fi
      ;;
  esac
}

usage() {
  cat <<EOF
usage: ${0##*/} [options] root

  Options:
    -L             Use labels for source identifiers (shortcut for -t LABEL)
    -p             Exclude pseudofs mounts (default behavior)
    -P             Include printing mounts
    -t TAG         Use TAG for source identifiers
    -U             Use UUIDs for source identifiers (shortcut for -t UUID)

    -h             Print this help message

genfstab generates output suitable for addition to an fstab file based on the
devices mounted under the mountpoint specified by the given root.

EOF
}

if [[ -z $1 || $1 = @(-h|--help) ]]; then
  usage
  exit $(( $# ? 0 : 1 ))
fi

while getopts ':LPpt:U' flag; do
  case $flag in
    L)
      bytag=LABEL
      ;;
    U)
      bytag=UUID
      ;;
    P)
      pseudofs=1
      ;;
    p)
      pseudofs=0
      ;;
    t)
      bytag=${OPTARG^^}
      ;;
    :)
      die '%s: option requires an argument -- '\''%s'\' "${0##*/}" "$OPTARG"
      ;;
    ?)
      die '%s: invalid option -- '\''%s'\' "${0##*/}" "$OPTARG"
      ;;
  esac
done
shift $(( OPTIND - 1 ))

(( $# )) || die "No root directory specified"
root=$(realpath -mL "$1"); shift

if ! mountpoint -q "$root"; then
  die "$root is not a mountpoint"
fi

# handle block devices
findmnt -Recvruno SOURCE,TARGET,FSTYPE,OPTIONS,FSROOT "$root" |
    while read -r src target fstype opts fsroot; do
  if (( !pseudofs )) && fstype_is_pseudofs "$fstype"; then
    continue
  fi

  # default 5th and 6th columns
  dump=0 pass=2

  src=$(unmangle "$src")
  target=$(unmangle "$target")
  target=${target#$root}

  if (( !foundroot )) && findmnt "$src" "$root" >/dev/null; then
    # this is root. we can't possibly have more than one...
    pass=1 foundroot=1
  fi

  # if there's no fsck tool available, then only pass=0 makes sense.
  if ! fstype_has_fsck "$fstype"; then
    pass=0
  fi

  if [[ $fsroot != / ]]; then
    if [[ $fstype = btrfs ]]; then
      opts+=,subvol=${fsroot#/}
    else
      # it's a bind mount
      src=$(findmnt -funcevo TARGET "$src")$fsroot
      if [[ $src -ef $target ]]; then
	# hrmm, this is weird. we're probably looking at a file or directory
	# that was bound into a chroot from the host machine. Ignore it,
	# because this won't actually be a valid mount. Worst case, the user
	# just re-adds it.
	continue
      fi
      fstype=none
      opts+=,bind
      pass=0
    fi
  fi

  # filesystem quirks
  case $fstype in
    fuseblk)
      # well-behaved FUSE filesystems will report themselves as fuse.$fstype.
      # this is probably NTFS-3g, but let's just make sure.
      if ! newtype=$(lsblk -no FSTYPE "$src") || [[ -z $newtype ]]; then
	# avoid blanking out fstype, leading to an invalid fstab
	error 'Failed to derive real filesystem type for FUSE device on %s' "$target"
      else
	fstype=$newtype
      fi
      ;;
  esac

  optstring_apply_quirks "opts" "$fstype"

  # write one line
  write_source "$src"
  printf '\t%-10s' "/$(mangle "${target#/}")" "$fstype" "$opts"
  printf '\t%s %s' "$dump" "$pass"
  printf '\n\n'
done

# handle swaps devices
{
  # ignore header
  read

  while read -r device type _ _ prio; do
    options=defaults
    if [[ $prio != -1 ]]; then
      options+=,pri=$prio
    fi

    # skip files marked deleted by the kernel
    [[ $device = *'\040(deleted)' ]] && continue

    if [[ $type = file ]]; then
      printf '%-20s' "$device"
    elif [[ $device = /dev/dm-+([0-9]) ]]; then
      # device mapper doesn't allow characters we need to worry
      # about being mangled, and it does the escaping of dashes
      # for us in sysfs.
      write_source "$(dm_name_for_devnode "$device")"
    else
      write_source "$(unmangle "$device")"
    fi

    printf '\t%-10s\t%-10s\t%-10s\t0 0\n\n' 'none' 'swap' "$options"
  done
} </proc/swaps

# vim: et ts=2 sw=2 ft=sh:
```