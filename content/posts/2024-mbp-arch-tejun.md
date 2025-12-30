---
title: "tech memo: Arch Linux MBP2018 installation with encryption"
tags: ["techmemo"]
date: 2024-11-24T19:43:22+09:00
---

Copy of <https://github.com/aidatorajiro/misc/blob/master/mbp-arch-tejun2024.md>

## resize ssd drive to make room for linux installation

Create **two partitions**, aside from mac os installation: (1) a 10GB luks-ext4 encrypted boot partition (2) the rest of disk will be the main luks-lvm partition

**Please backup EFI system partition (using `dd` or `rsync`) and the main Mac OS installation and Mac OS user files (using time machine) before booting USB drive.**

## download t2linux iso & boot

Download the ISO from here:

https://github.com/t2linux/archiso-t2/releases

copy it to a USB drive (using `dd` command), and boot the USB drive.

## inside linux iso...

### set envs to avoid accident

!!!!!!!! please double check device names, everything on MAINBOOT and MAINVG will be erased !!!!!!!!

```bash
echo MAINEFI=/dev/nvme0n1p1 >> myenvs
echo MAINBOOT=/dev/nvme0n1p3 >> myenvs
echo MAINVG=/dev/nvme0n1p4 >> myenvs
. ./myenvs
```

### format VG & BOOT

```bash
cryptsetup luksFormat --type luks1 $MAINBOOT
cryptsetup luksFormat $MAINVG
cryptsetup open $MAINVG cryptvg
cryptsetup open $MAINBOOT cryptboot
mkfs.ext4 /dev/mapper/cryptboot
vgcreate mbpvg /dev/mapper/cryptvg
lvcreate -L 100G -n root mbpvg
lvcreate -L 100G -n home mbpvg
mkfs.ext4 /dev/mapper/mbpvg-root 
mkfs.ext4 /dev/mapper/mbpvg-home
mount /dev/mapper/mbpvg-root /mnt
mkdir /mnt/home /mnt/boot
mount /dev/mapper/mbpvg-home /mnt/home
mount /dev/mapper/cryptboot /mnt/boot
mkdir /mnt/boot/efi
mount $MAINEFI /mnt/boot/efi
```

### connect to wifi using iwctl
```bash
iwctl
station wlan0 get-networks
station wlan0 connect [SOMESSID]
exit
```

### add t2-linux repo
```bash
vim /etc/pacman.conf
```

append these lines:
```toml
[arch-mact2]
Server = https://mirror.funami.tech/arch-mact2/os/x86_64
SigLevel = Never
```

### pacstrap to install necessary items
```bash
pacstrap /mnt base linux-t2 apple-t2-audio-config apple-bcm-firmware iwd grub efibootmgr tiny-dfr t2fanrd linux-firmware iwd networkmanager vim archlinux-keyring sudo less ripgrep lvm2 bluez blueman usbutils
```

### add t2-linux repo again
```bash
vim /mnt/etc/pacman.conf
```

append these lines:
```toml
[arch-mact2]
Server = https://mirror.funami.tech/arch-mact2/os/x86_64
SigLevel = Never
```

### genfstab and chroot
```bash
genfstab -U /mnt >> /mnt/etc/fstab
lsblk -o uuid,name > /mnt/lsblkresult
vim /mnt/lsblkresult # edit lsblkresult to make some UUID database that can be parsed by bash
cp myenvs /mnt
arch-chroot /mnt
```

lsblkresult should look like this:
```bash
# UUID                                   NAME
        #                               loop0
#XXXX-XX-XX-XX-XX-XX-XX                 sda
#XXXX-XX-XX-XX-XX-XX-XX                 ├─sda1
#XXXX-XXXX                              └─sda2
#                                       nvme0n1
EFIUUID=XXXX-XXXX  #                            ├─nvme0n1p1
#XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   ├─nvme0n1p
BOOTUUID1=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX #  ├─nvme0n1p3
BOOTUUID2=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX #  │ └─cryptboot
VGUUID1=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX #  └─nvme0n1p4
VGUUID2=xXxXxX-xXxX-xXxX-xXxX-xXxX-xXxX-xXxXxX  # └─cryptvg
ROOTUUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX    #   ├─mbpvg-root
HOMEUUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX     #  └─mbpvg-home
```

## inside chroot

### import keys
```bash
pacman-key --init
pacman-key --populate
```

### config something
```bash
systemctl enable t2fanrd
ln -sf /usr/share/zoneinfo/[SOMEPLACE] /etc/localtime
hwclock --systohc
vim /etc/locale.gen # uncomment language(s) you use
vim /etc/locale.conf # set LANG=xx_XX.UTF-8
locale-gen
vim /etc/hostname # set hostname
useradd -m someuser
passwd someuser
EDITOR=vim visudo # add to sudoers
su someuser
sudo ls # test if sudo works
exit
```

### config mkinitcpio
```bash
vim /etc/mkinitcpio.conf
```

replace these values:
```toml
MODULES=(apple-bce)
FILES=(/root/bootkey)
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)
```

### generate secret to unlock disks
```bash
cd /root
dd bs=512 count=4 if=/dev/random of=/root/bootkey iflag=fullblock
chmod 000 bootkey
. /myenvs
. /lsblkresult
cryptsetup luksAddKey $MAINBOOT /root/bootkey 
cryptsetup luksAddKey $MAINVG /root/bootkey
```

### config grub
```bash
vim /etc/default/grub
```

replace these values:
```toml
GRUB_CMDLINE_LINUX_DEFAULT="root=UUID=ROOTUUID cryptkey=rootfs:/root/bootkey cryptdevice=UUID=VGUUID:cryptvg loglevel=3 quiet intel_iommu=on iommu=pt pcie_ports=compat"
GRUB_ENABLE_CRYPTODISK=y
```

rootuuid and vguuid **MUST** be replaced with actual uuid

```bash
sed -i.bak s/VGUUID/$VGUUID1/ /etc/default/grub # replace with uuid of encrypted boot
sed -i.bak s/ROOTUUID/$ROOTUUID/ /etc/default/grub # replace with uuid of the root volume inside the vg
```

### edit /etc/crypttab to unlock /boot after the linux is up
```bash
vim /etc/crypttab
```

just put this line:
```
boot	UUID=BOOTUUID	/root/bootkey
```

and replace with the uuid of encrypted boot
```bash
sed -i.bak s/BOOTUUID/$BOOTUUID1/ /etc/crypttab
```

### generate vmlinuz. initramfs, grub, grub cfg
```bash
mkinitcpio -P
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB --removable
grub-mkconfig -o /boot/grub/grub.cfg
```

### config NetworkManager (important because iwd doesn't setup dhcp and routing by default)

```bash
vim /etc/NetworkManager/conf.d/iwd.conf
```

```toml
[device]
wifi.backend=iwd
wifi.iwd.autoconnect=yes
```

alternatively, without NetworkManager, you can create /etc/iwd/main.conf as follows:
```toml
[General]
EnableNetworkConfiguration=true
```

### (if you are using NetworkManager) optionally, you can add Network Address Randomization feature

Create a file `/etc/iwd/main.conf` as follows to enable address randomization feature, <s>which (as far as i know) is not supported on usual Mac OS!<s> (Update May 2025: i found the "Private Wi-Fi address" option in the "Network Settings..." panel.)

```toml
[General]
AddressRandomization=network
```

Then, create some file like `/etc/NetworkManager/conf.d/randmac.conf` and set the contents as follows:

```toml
[device]
wifi.scan-rand-mac-address=yes
 
[connection]
wifi.cloned-mac-address=stable
ipv6.dhcp-duid=stable-uuid
connection.stable-id=abcdefsomerandomtext-${CONNECTION}
```

this will keep the mac address for the same wifi network. replace `abcdefsomerandomtext` with random token you like, such as a randomly generated UUID. You may want to set up a cron that regenerates this file every week or so.


## exit chroot and reboot

```bash
exit
reboot
```

## note on usb boot

linux's xhci_hcd driver is somewhat broken, so if you are going to usb boot, you should write `options usb-storage quirks=XXXX:XXXX:u` on `/etc/modprobe.d/usbfix.conf`, where `XXXX:XXXX` is the vender and product id obtained by `lsusb`.

for example, config for RTL9210 will be `options usb-storage quirks=0bda:9210:u`
