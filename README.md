# openwrt-snapcast-a5-v11
Snapclient port to cheap chinese router known as a5-v11.

Work heavily based on https://github.com/ozayturay/OpenWrt-A5-V11

Target OpenWrt version: 18.06.9


# Firmware build instructions
    make image PROFILE="a5-v11" PACKAGES="-luci block-mount kmod-nls-cp437 kmod-nls-iso8859-1 kmod-nls-utf8 kmod-fs-vfat kmod-fs-ext4 kmod-usb-storage-extras kmod-usb2 nano -firewall -ip6tables -kmod-ip6tables -kmod-ipv6 -odhcp6c -ppp -ppp-mod-pppoe"

# Initial setup (original firmware)
Based on instructions here: https://github.com/ozayturay/OpenWrt-A5-V11

Use a flash drive formatted to FAT32, copy uboot256 and firmware file (must end in factory) there. Plug the flash drive to the mini router and connect to it's WiFi network. Telnet to 192.168.100.1 (you need to install telnet on newer Windows).
Credentials are admin/admin.

Run these commands, one by one:

    umount /dev/sda1
    mount /dev/sda1 /mnt
    mtd_write write /mnt/uboot256.img Bootloader
    mtd_write write /mnt/openwrt-18.06.9-ramips-rt305x-a5-v11-squashfs-factory.bin Kernel

Writing the firmware file takes some time, so be patient. After it is done, you can reboot:

    reboot

Now the defaults for OpenWrt image will create a network 192.168.1.1 with no DHPC server, so connect the ethernet cable directly to some device and assign yourself IP 192.168.1.2.

Telnet to 192.168.1.1 and set passwd to root user. After it is done, SSH connection should now be available. After connecting with SSH (still on 192.168.1.1), change network config:

    nano /etc/config/network

Change line

    network.lan.proto="static"

to

    network.lan.proto="dhcp"

Save and reboot the device, connect it to your local network, find out IP address of it and connec with SSH again. Ping some url to make sure the device can reach the internet before you proceed.

## Extend available space

Prepare USB flash drive to extend available space on the device - pretty much any size will work.

    Overlay (1 GiB): primary boot ext4
    Swap (512 MiB): primary swap
    Data (Rest): primary ext4

Plug in the drive and check on the router whether it was recognized:

    cd /dev/
    ls -la

You should be able to see /dev/sda1. If not, refer to this guide to install additional packages: https://openwrt.org/docs/guide-user/storage/usb-drives We don't have much space to spare, so don't install everything without thinking.
If the flash drive is there, we should perform extroot.

    block detect > /etc/config/fstab; \
     sed -i s/option$'\t'enabled$'\t'\'0\'/option$'\t'enabled$'\t'\'1\'/ /etc/config/fstab; \
     sed -i s#/mnt/sda1#/overlay# /etc/config/fstab; \
     sed -i s#/mnt/sda3#/data# /etc/config/fstab; \
     cat /etc/config/fstab;

Configure opkg and create the directory:

    echo option force_space >> /etc/opkg.conf
    mkdir /data

Mount and copy overlay:

    umount /dev/sda1
    mount /dev/sda1 /mnt
    tar -C /overlay -cvf - . | tar -C /mnt -xf -

Now you can reboot - after reboot you can check the available space and more importantly, install new packages:

    opkg update
    opkg install luci

Don't overdo it, 360Mhz can run only so much stuff.

# Snapclient installation

Download prebuilt packages from this repository - at the very least you need boost and snapclient to make it work. Copy all the packages to the /data directory, you can just install SFTP server if you want.

    opkg install openssh-sftp-server

Now install packages:

    opkg install /data/package.ipk

Installation of snapclient would download load of other packages - if you install it first, make sure to upgrade boost from this repository - it is required by this version of snapclient!
Configure snapclient and hope for the best:
      
    nano /etc/default/snapclient
    SNAPCLIENT_OPTS="-d -h <snapserver-ip> -s 3"

# Upgrading from older OpenWrt version
Use the supplied sysupgrade file.

# Future and FAQ

## Snapclient version is only 0.26.0
This is the latest version I was able to build for now - seems that the newest version needs C++17 stuff that I'm not able to build for 18.06.9 image. For now, version 0.27.0 doesn't have any important client fixes.

## OpenWrt 18.06.9 is not supported
This is the last version "officially" supported on A5-V11 - there is a chance that version 19 could also work - right now I'm not working on that. Please don't expose this device to the Internet.

## I want to try to build it myself
I plan to write down complete instructions, for now I will just mark down some stuff that I had to fix:

Build system is Ubuntu 22.04 - use SnapOS instructions - but checkout the commit 9763d9a to build older version.
Build problem fixes: https://git.freifunk-franken.de/mirror/openwrt/commit/a1ee0ebbd8e9927a65c5d1e0db497dd118d559a6
Use boost Makefile from OpenWrt version 21 and up - otherwise snapcast build will fail because the boost in 18.06.9 is too old. 
