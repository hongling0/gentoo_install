#安装Gentoo流程记录

#ultraiso写入硬盘镜像到U盘，用U盘启动

#磁盘分区
fdisk /dev/sda

### /dev/sda1 boot分区
### /dev/sda2 swap分区
### /dev/sda3 /分区
### /dev/sda5 逻辑分区

# 创建LVM分区

pvcreate /dev/sda5

vgcreate vg /dev/sda5

lvcreate -L8G -nusr vg

lvcreate -L8G -nportage vg

lvcreate -L8G -nvartmp vg

lvcreate -L6G -ndistfiles vg

lvcreate -L6G -nopt vg

lvcreate -L6G -nvar vg

lvcreate -L6G -ntmp vg

lvcreate -L9G -nhome vg

#格式化
mke2fs /dev/sda1

mke2fs -j /dev/sda2

mke2fs -b 4096 -T largefile /dev/vg/distfiles

mke2fs -j /dev/vg/home

mke2fs -j /dev/vg/opt

mke2fs -b 1024 -N 200000 /dev/vg/portage

mke2fs /dev/vg/tmp

mke2fs -j /dev/vg/usr

mke2fs -j /dev/vg/var

mke2fs /dev/vg/vartmp

#加载交换分区
mkswap /dev/sda3

swapon -p 1 /dev/sda3

swapon -v -s

#mount
mount /dev/sda2 /mnt/gentoo

cd /mnt/gentoo

mkdir boot home usr opt var tmp

mount /dev/sda1 /mnt/gentoo/boot

mount /dev/vg/usr /mnt/gentoo/usr

mount /dev/vg/home /mnt/gentoo/home

mount /dev/vg/opt /mnt/gentoo/opt

mount /dev/vg/tmp /mnt/gentoo/tmp

mount /dev/vg/var /mnt/gentoo/var

mkdir usr/portage var/tmp

mount /dev/vg/vartmp /mnt/gentoo/var/tmp

mount /dev/vg/portage /mnt/gentoo/usr/portage

mkdir usr/portage/distfiles

mount /dev/vg/distfiles /mnt/gentoo/usr/portage/distfiles

chmod 1777 /mnt/gentoo/tmp /mnt/gentoo/var/tmp

#解压安装包
tar xjpf stage3*

#chroot
cd /
mount -t proc proc /mnt/gentoo/proc

mount --rbind /dev /mnt/gentoo/dev

mount --rbind /sys /mnt/gentoo/sys

cp -L /etc/resolv.conf /mnt/gentoo/etc/ 

chroot /mnt/gentoo /bin/bash

source /etc/profile

#初始化portage树
mkdir -p /usr/portage

echo 'GENTOO_MIRRORS="http://mirrors.163.com/gentoo/"' >> /etc/portage/make.conf
echo 'SYNC="rsync://rsync2.cn.gentoo.org/gentoo-portage"' >> /etc/portage/make.conf

emerge-webrsync

#设置时区
ls /usr/share/zoneinfo

cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

echo "Asia/Shanghai" > /etc/timezone

date

#选择profile
eselect profile list

eselect profile set 1

#设置hostname
cd /etc
echo "127.0.0.1 gentoo.at.linux gentoo localhost" > hosts

sed -i -e 's/hostname.*/hostname="gentoo"/' conf.d/hostname

hostname gentoo

hostname -f

#编译内核
emerge gentoo-sources

cd /usr/src/linux

make menuconfig

make -j3

make modules_install

cp arch/i386/boot/bzImage /boot/kernel`uname -r`

#配置make.conf
emerge ccache mirrorselect

mirrorselect -i -o >> /etc/portage/make.conf

mirrorselect -i -r -o >> /etc/portage/make.conf

echo 'MAKEOPTS="-j3"' >> /etc/portage/make.conf

echo 'CCACHE_DIR="/var/tmp/ccache"' >>/etc/portage/make.conf

echo 'CCACHE_SIZE="2G"' >>/etc/portage/make.conf

echo 'FEATURES="ccache userfetch parallel-fetch"' >>/etc/portage/make.conf

#编译initramfs
emerge genkernel
genkernel --install --no-ramdisk-modules --lvm --mdadm initramfs

#编写 fstab
cd /etc

cat > fstab << EOF

/dev/sda1          /boot                   ext2  noauto,noatime  1 2

/dev/sda2          /                       ext3  noatime         0 1

/dev/sda3         none                    swap  sw,pri=1        0 0

/dev/vg/usr       /usr                    ext3  noatime         1 2

/dev/vg/portage   /usr/portage            ext2  noatime         1 2

/dev/vg/distfiles /usr/portage/distfiles  ext2  noatime         1 2

/dev/vg/home      /home                   ext3  noatime         1 2

/dev/vg/opt       /opt                    ext3  noatime         1 2

/dev/vg/tmp       /tmp                    ext2  noatime         1 2

/dev/vg/var       /var                    ext3  noatime         1 2

/dev/vg/vartmp    /var/tmp                ext2  noatime         1 2
EOF

#设置网卡启动 ifconfig | grep flag | awk -F: '{print $1}' | grep -v lo
cd init.d

ln -s net.lo net.enp0s3

cd ../conf.d

rc-update add net.enp0s3 default

#sshd
rc-update add sshd default

#设置密码
passwd

#安装 raid lvm log cron
emerge mdadm lvm2 syslog-ng vixie-cron dhcpcd grub

rc-update add mdraid boot

rc-update add lvm boot

rc-update add syslog-ng default

rc-update add vixie-cron default

grub2-install /dev/sda

grub2-mkconfig -o /boot/grub/grub.cfg

#退出重启
exit
umount -l /mnt/gentoo/usr{/portage/distfiles,/portage,}

umount -l /mnt/gentoo/dev{/pts,/shm,}

umount -l /mnt/gentoo{/usr,/var/tmp,/tmp,/var,/opt,/dev,/proc,/home,/boot,}

reboot

#创建帐号
useradd -g users -G lp,wheel,audio,cdrom,portage,cron -m hong

passwd hong

#本地化
cd /etc

nano -w locale.gen

locale-gen

#安装常用软件
emerge gentoolkit vim sudo git app-shells/gentoo-bashcomp fontconfig

eselect bashcomp enable git

eselect bashcomp enable gentoo

