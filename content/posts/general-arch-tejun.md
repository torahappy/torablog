---
title: "tech memo: general Arch Linux installation with encryption w/signing"
tags: ["techmemo"]
date: 2025-01-01T03:10:00+09:00
---

## resize ssd drive to make room for linux installation

Create **two partitions**, aside from windows installation: (1) a 10GB FAT32 EFI system partition (2) the rest of disk will be the main luks-lvm partition.

...or you can delete pre-existing windows installation and use `fdisk` to create partitions after booting Arch ISO.

**Please backup EFI system partition (using `dd` or `rsync`) and the main Windows installation and user files before booting USB drive.**

## download linux iso & boot

Download the ISO from here:

<https://archlinux.org/download/>

copy it to a USB drive (using `dd` command or Rufus <https://rufus.ie/>), and boot the USB drive.

Then, turn OFF the Secure Boot.

## inside linux iso...

### backup partition table
backup old EFI keys using following command. it is advised to copy them to an external drive.

```bash
# insert another usb drive (assuming /dev/sdb2)
mount /dev/sdb2 /mnt
cd /mnt
mkdir old-system-backup
cd old-system-backup
# list available disks
ls /dev/nvme* /dev/sd*
# backup partition tables
fdisk -l /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 ... /dev/sda /dev/sdb /dev/sdc ... > partition-tables
# check output
less partition-tables
cd /
umount /mnt
```

### set envs to avoid accident

!!!!!!!! please double check device names, everything on MAINBOOT and MAINVG will be erased !!!!!!!!

```bash
# check partition tables
fdisk -l /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 ...

# choose the partitions to erase
echo MAINEFI=/dev/nvmexnxpx >> myenvs
echo MAINVG=/dev/nvmexnxpx >> myenvs
. ./myenvs
```

### format VG & BOOT

```bash
cryptsetup luksFormat $MAINVG
cryptsetup open $MAINVG cryptvg
vgcreate linuxvg /dev/mapper/cryptvg
lvcreate -L 100G -n root linuxvg
lvcreate -L 100G -n home linuxvg
mkfs.ext4 /dev/mapper/linuxvg-root 
mkfs.ext4 /dev/mapper/linuxvg-home
mount /dev/mapper/linuxvg-root /mnt
mkdir /mnt/home /mnt/boot
mount /dev/mapper/linuxvg-home /mnt/home
mount $MAINEFI /mnt/boot
mkdir /mnt/boot/EFI
```

### connect to wifi using iwctl

the device name `wlan0` may be different (specific to the machine you are using)

```bash
iwctl
station wlan0 get-networks
station wlan0 connect [SOMESSID]
exit
```

### pacstrap to install necessary items
```bash
pacstrap /mnt base linux iwd efibootmgr linux-firmware networkmanager vim neovim archlinux-keyring sudo less ripgrep lvm2 bluez blueman usbutils efitools sbctl
```

### genfstab and chroot
```bash
genfstab -U /mnt >> /mnt/etc/fstab
lsblk -o uuid,name > /mnt/lsblkresult
nvim /mnt/lsblkresult # edit lsblkresult to make some UUID database that can be parsed by bash
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
#XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX           ├─nvmexnxpx
VGUUID1=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX #  └─nvmexnxpx
VGUUID2=xXxXxX-xXxX-xXxX-xXxX-xXxX-xXxX-xXxXxX  # └─cryptvg
ROOTUUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX    #   ├─linuxvg-root
HOMEUUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX     #  └─linuxvg-home
```

## inside chroot

### backup EFI variables
```bash
mkdir /mnt
mount /dev/sdb2 /mnt # an external drive to backup files
cd /mnt
mkdir ./efivar-backup
cd ./efivar-backup
for var in PK KEK db dbx ; do efi-readvar -v $var -o old_${var}.esl ; done
cd /
umount /mnt
```

### import keys
```bash
pacman-key --init
pacman-key --populate
```

### config something
```bash
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

### config mkinitcpio (HOOKS)
```bash
vim /etc/mkinitcpio.conf
```

replace these values:
```toml
HOOKS=(microcode base udev autodetect modconf keyboard keymap consolefont block encrypt lvm2 filesystems fsck)
```

### config cmdline
```bash
echo "root=UUID=ROOTUUID rw lsm=landlock,lockdown,yama,integrity,apparmor,bpf cryptdevice=UUID=VGUUID:cryptvg loglevel=3 quiet" > /etc/cmdline.d/root.conf
```

rootuuid and vguuid **MUST** be replaced with actual uuid

```bash
sed -i.bak s/VGUUID/$VGUUID1/ /etc/cmdline.d/root.conf # replace with uuid of encrypted boot
sed -i.bak s/ROOTUUID/$ROOTUUID/ /etc/cmdline.d/root.conf # replace with uuid of the root volume inside the vg
```

### config mkinitcpio (preset)

```bash
vim /etc/mkinitcpio.d/linux.preset
```

Replace with these values:

```bash
# mkinitcpio preset file for the 'linux' package

#ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux"

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux.img"
default_uki="/boot/EFI/BOOT/BOOTx64.EFI"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

#fallback_config="/etc/mkinitcpio.conf"
#fallback_image="/boot/initramfs-linux-fallback.img"
fallback_uki="/boot/EFI/FLBK/FLBKx64.EFI"
fallback_options="-S autodetect"
```

### prepare the signing key, and generate the main EFI
```bash
sbctl create-keys
sbctl sign -s /boot/EFI/BOOT/BOOTx64.EFI
sbctl sign -s /boot/EFI/FLBK/FLBKx64.EFI
mkinitcpio -P
```

### Config Network

Choose from either NetworkManager or NetworkManager+iwd.

### config NetworkManager only

```bash
systemctl enable NetworkManager
systemctl disable iwd
```

### Or, config NetworkManager+iwd

```bash
systemctl enable NetworkManager
systemctl enable iwd
```

Create a file `/etc/NetworkManager/NetworkManager.conf` as follows:

```toml
[device]
wifi.backend=iwd
wifi.iwd.autoconnect=yes
```

Create a file `/etc/iwd/main.conf` as follows to enable address randomization feature.

```toml
[General]
AddressRandomization=network
```

### Config MAC Address Randomization

Create some file like `/etc/NetworkManager/conf.d/randmac.conf` and set the contents as follows:

```toml
[device]
wifi.scan-rand-mac-address=yes
 
[connection]
wifi.cloned-mac-address=stable
ipv6.dhcp-duid=stable-uuid
connection.stable-id=abcdefsomerandomtext-${CONNECTION}
```

this will keep the mac address for the same wifi network. replace `abcdefsomerandomtext` with random token you like, such as a randomly generated UUID. You may want to set up a cron that regenerates this file every week or so.

### export the public keys to EFI partition

Copy public keys to the EFI partition. Do not export `*.key` files, these files are confidential.

```bash
mkdir /boot/exported-keys
cd /boot/exported-keys
cp /var/lib/sbctl/keys/db/db.pem /var/lib/sbctl/keys/KEK/KEK.pem /var/lib/sbctl/keys/PK/PK.pem .
openssl x509 -outform der -in db.pem -out db.der
openssl x509 -outform der -in KEK.pem -out KEK.der
openssl x509 -outform der -in PK.pem -out PK.der
```

## exit chroot and reboot

```bash
exit
reboot
```

Please use the your machine's boot manager to launch the Linux Parition. ( Such as pressing F11 right after boot. The procedure varies depending on the machine you use. )

Check if you can boot the system and have access to terminal, login, run commands as root etc. You can install desktop + desktop manager at this point.

For example, in the case of XFCE4+lightdm, run `sudo pacman -Sy xfce4 lightdm lightdm-gtk-greeter && sudo systemctl enable lightdm`. Then, edit `/etc/lightdm/lightdm.conf`, replacing following value:

```toml
greeter-session=lightdm-gtk-greeter
```

Then, run `sudo systemctl start lightdm` to check the graphical interface.

## Setup Secure Boot Signing

Although you have already signed the EFI at this point by installing `sbctl`, the signature key have to be uploaded to your machine.

You can also setup shim, which may be already signed by the machine's preinstalled key (Microsoft's 3rd Party Signing Key), but this depends on machines. If you use shim, you can skip the Secure Boot setup phase but instead you have to enroll your key to MOK using shared passphrase instead.

In this article, we upload the signature to the machine directly (not using shim and MOK).

First, open your machine's UEFI Settings screen (such as pressing DEL right after the setup.)
Then, turn off fast boot, rearrange boot priority, enable Secure Boot and **importantly set the Secure Boot to Setup mode**. You may have to disable setup key provisioning to avoid making Setup unusable.

Alternatively, you may be able to configure keys in UEFI Settings screen. If you can do so, just enroll the keys `db.der` and/or `KEK.der` and/or `PK.der` and/or factory default (Windows) keys, instead of using Setup mode.

After enabling setup mode, just boot the Linux EFI. 

### check setup mode status & enroll keys

If you fail to boot (e.g. invalid signature), you are not in setup mode. You may have to disable key provisioning in the UEFI settings screen and enter setup mode again.

Check Secure Boot Status. Ensure Setup mode is enabled.

```bash
sudo sbctl status
```

Enroll keys to the Machine, along with Microsoft keys.

```bash
sudo sbctl sign -s /boot/EFI/BOOT/BOOTx64.EFI
sudo sbctl sign -s /boot/EFI/FLBK/FLBKx64.EFI
sudo sbctl enroll-keys -m
sudo mkinitcpio -P
sudo sbctl verify
sudo reboot
```

If Linux can boot after rebooting, installation is successful!
