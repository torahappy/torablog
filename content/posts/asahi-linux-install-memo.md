---
title: "asahi linux install memo"
tags: []
date: 2025-12-10T02:39:58+09:00
---

(1TB SSD setting)

please backup all data before carrying out the process below.

Please be extra careful for numbers around partitioning! (such as disk indices, sector start/end/length) these numbers may be different on your environment!!!

## macos: asahi setup

Follow instructions at https://asahilinux.org/

```bash
curl https://alx.sh | sh
```

Specify installation options as follows:
```
r: Resize an existing partition to make space for a new OS
    50%
f: Install an OS into free space
4: Fedora Asahi Remix 42 Minimal
» New OS size (max): 40GB
```

Reboot and setup Asahi.

## asahi: first asahi boot

install wifi credentials, basic apps, font settings, updates

```bash
sudo nmcli device wifi # show ssids
sudo nmcli device wifi connect MYSSID --ask
sudo dnf install git w3m tmux terminus-fonts-console neovim xxd
setfont ter-v32n
echo "FONT=ter-v32n" | sudo tee -a /etc/vconsole.conf 
sudo dnf update
reboot
```

## macos: change asahi disk position & backup original disk to somewhere

Check the partition layout
```bash
diskutil list
```

Results:
```
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *1.0 TB     disk0
   1:             Apple_APFS_ISC Container disk1         524.3 MB   disk0s1
   2:                 Apple_APFS Container disk4         497.3 GB   disk0s2
   3:                 Apple_APFS Container disk3         2.5 GB     disk0s3
   4:                        EFI EFI - FEDOR             524.3 MB   disk0s4
   5:           Linux Filesystem                         1.1 GB     disk0s5
   6:           Linux Filesystem                         35.9 GB    disk0s6
                    (free space)                         457.3 GB   -
   7:        Apple_APFS_Recovery Container disk2         5.4 GB     disk0s7
```

Create new Partition (similar size as disk0s6, the original asahi root)

Query for the exact size:
```
diskutil list -plist | rg -C10 disk0s6
```

Results:
```
				<dict>
					<key>Content</key>
					<string>Linux Filesystem</string>
					<key>DeviceIdentifier</key>
					<string>disk0s6</string>
					<key>DiskUUID</key>
					<string>XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX</string>
					<key>Size</key>
					<integer>35901145088</integer>
				</dict>
```

Size=35901145088
 →Create 37GB Volume

Create 37GB volume after disk0s6 (the original asahi linux root)
```
diskutil addPartition disk0s6 JHFS+ MyVol 37GB
```

Results:
```
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *1.0 TB     disk0
   1:             Apple_APFS_ISC Container disk1         524.3 MB   disk0s1
   2:                 Apple_APFS Container disk4         497.3 GB   disk0s2
   3:                 Apple_APFS Container disk2         2.5 GB     disk0s3
   4:                        EFI EFI - FEDOR             524.3 MB   disk0s4
   5:           Linux Filesystem                         1.1 GB     disk0s5
   6:           Linux Filesystem                         35.9 GB    disk0s6
   7:                  Apple_HFS MyVol                   36.9 GB    disk0s9
                    (free space)                         420.5 GB   -
   8:        Apple_APFS_Recovery Container disk3         5.4 GB     disk0s7
```

You might get disk0s8 as your new partition. If that happens, change numbers below accordingly.

Make sure size of disk0s6 (original asahi linux root) < size of disk0s9 (new partition)

```
diskutil list -plist | rg -C10 disk0s9
```

```
					<key>Size</key>
					<integer>36865781760</integer>
```

```
diskutil list -plist | rg -C10 disk0s6
```

```
					<key>Size</key>
					<integer>35901145088</integer>
```

Backup the disk to an external drive

```
sudo dd if=/dev/disk0s6 of=/Volumes/mount-1/disk0s6-copy bs=10M status=progress
```

Copy the disk

```
sudo dd if=/dev/disk0s6 of=/dev/disk0s9 bs=10M status=progress
```

Delete original
```
sudo diskutil eraseVolume JHFS+ UntitledHFS /dev/disk0s6
```

Reboot and continue setup.

## asahi: second asahi boot

mount external drive

```bash
sudo mount /dev/sda2 mount
sudo systemctl daemon-reload
```

show current partition table

```bash
sudo fdisk -l /dev/nvme0n1
```

example output
```
Disk /dev/nvme0n1: 931.84 GiB, 1000555581440 bytes, 244276265 sectors
Disk model: APPLE SSD                       
Units: sectors of 1 * 4096 = 4096 bytes
Sector size (logical/physical): 4096 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: XXXXXXXX-XXXX-XXXXX-XXXX-XXXXXXXXXXXX

Device             Start       End   Sectors   Size Type
/dev/nvme0n1p1         6    128005    128000   500M Apple Silicon boot
/dev/nvme0n1p2    128006 121547013 121419008 463.2G Apple APFS
/dev/nvme0n1p3 121547014 122157317    610304   2.3G Apple APFS
/dev/nvme0n1p4 122157318 122285317    128000   500M EFI System
/dev/nvme0n1p5 122285318 122547461    262144     1G Linux filesystem
/dev/nvme0n1p6 122547462 131279621   8732160  33.3G Apple HFS/HFS+
/dev/nvme0n1p7 131312390 140312824   9000435  34.3G Apple HFS/HFS+
/dev/nvme0n1p8 242965551 244276259   1310709     5G Apple Silicon recovery
```

In this case,
```
/dev/nvme0n1p6                 = encrypted root (destination)
/dev/nvme0n1p7                 = filesystem used for current root
some file in an external drive = source files to copy to encrypted root
```

format 6 (destination partition) with LUKS encryption

```bash
sudo cryptsetup luksFormat /dev/nvme0n1p6
sudo cryptsetup open /dev/nvme0n1p6 MainDecrypted
```

copy the original disk data to the destination partition
 
```bash
# create new btrfs
sudo mkfs.btrfs /dev/mapper/MainDecrypted
sudo mount /dev/mapper/MainDecrypted /mnt
sudo btrfs subvolume create /mnt/root
sudo btrfs subvolume create /mnt/home
sudo btrfs subvolume list /mnt
sudo umount /mnt

# copy the original data to the destination
sudo mount -o subvol=root /dev/mapper/MainDecrypted /mnt
sudo mount --mkdir -o subvol=home /dev/mapper/MainDecrypted /mnt/home

mkdir mount_source
sudo losetup /dev/loop100 ./mount/disk0s6-copy 
sudo mount -o subvol=root /dev/loop100 ./mount_source/
sudo mount -o subvol=home /dev/loop100 ./mount_source/home/

sudo rsync -avX ./mount_source/ /mnt/

sudo umount mount_source/home
sudo umount mount_source/
```

Chroot and run restorecon
```bash
sudo mount --rbind /boot /mnt/boot/
sudo mount --rbind /boot/efi/ /mnt/boot/efi/
sudo mount -t proc /proc /mnt/proc
sudo mount --rbind /sys /mnt/sys
sudo mount --rbind /dev /mnt/dev
sudo mount --bind /dev/pts /mnt/dev/pts
sudo chroot /mnt /bin/bash
restorecon -R .
```

### INSIDE CHROOT

#### Fix network

This links to ../run/systemd/resolve/stub-resolv.conf, so u better relink it after reboot.

Nonetheless, as NetworkManager will probably replace this file, systemd stub link won't be a thing if things are set up correctly. (like after you login graphically and connect WiFi via NetworkManager applet.)

```bash
sudo cp /etc/resolv.conf /etc/resolv.conf.bak
sudo rm /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee -a /etc/resolv.conf
```

#### edit auto-generation config and regenerate grubconf/initrd

sadly, lsblk could not print UUIDs in chroot environment :(
grub uses /dev/disk/by-uuid to search for UUIDs, so this doesn't cause any trouble...hopefully...

`ls -al /dev/disk/by-uuid/` lists all uuids and the corresponding device references as symbolic links. Those starts with /dev/nvme0n1 are the internal SSD drive partitions, and those starts with /dev/dm- are the decrypted data via cryptsetup/dm-crypt.

```bash
ls -al /dev/disk/by-uuid/ >> /etc/default/grub
sudo nvim /etc/default/grub 
```

replace GRUB_CMDLINE_LINUX_DEFAULT with `rd.luks.uuid=<uuid of the partition to decrypt>` (i.e. /dev/nvme0n1p6)

```bash
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_DISABLE_RECOVERY=true
GRUB_DISABLE_OS_PROBER=true
GRUB_CMDLINE_LINUX_DEFAULT="rd.luks.uuid=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX" 
GRUB_DISTRIBUTOR="Fedora Linux Asahi Remix"
GRUB_GFXMODE=auto
GRUB_TERMINAL_INPUT="console"
GRUB_TERMINAL_OUTPUT="gfxterm"
GRUB_TIMEOUT=5
GRUB_TIMEOUT_STYLE=menu
```

regenerate grub config

```
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

regen initrd

```
sudo dracut --force
```

#### update FSTAB ( *very important* if you fail to do this, your system won't start because there are no home partition!! )

link each mount option and (uu)id in /etc/fstab
```
uuid of /dev/dm-0      (encrypted root) <-> btrfs, / and /home
uuid of /dev/nvme0n1p5 (boot partition) <-> ext4, /boot
  id of /dev/nvme0n1p4 (EFI partition)  <-> vfat, /boot/efi
```

Edit fstab according to the fstab format
```
sudo ls -al /dev/disk/by-uuid/ | sudo tee -a /etc/fstab
sudo nvim /etc/fstab
```

Exit chroot, reboot and see if anything went wrong
```
exit
sudo reboot
```

### install desktop, etc.

install desktop

```
sudo dnf group install xfce-apps xfce-desktop
sudo dnf install lightdm lightdm-gtk-greeter
```

set lightdm-gtk-greeter as the greeter

```
sudo nvim /etc/lightdm/lightdm.conf
```

create a custom autostart script

```
sudo nvim /etc/systemd/system/my-autostart.service
```

set the content to `systemctl start lightdm` and the timing to `multi-user.target`

```
[Unit]
Description=My Custom Autostart for lightdm
After=multi-user.target

[Service]
ExecStart=/usr/bin/systemctl start lightdm
Type=simple

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl enable --now my-autostart.service
```

this will start lightdm immediately and will be persistent across reboots.

### expand /dev/nvme0n1p6 and delete /dev/nvme0n1p7

1. Memo the sector positions of 6 (main partition), 8 (recovery partition) and remove 6, 7, 8.
2. Create a new partition 6 and specify the exactly same start position as before, and expand the end sector (you should have about 2GB margin until the start of the recovery partition).
3. Create a new partition 8 and specify the exactly same start and end positions as before. This will be the new recovery partition. IMPORTANT: **SET THE TYPE OF THE RECOVERY PARTITION TO `APPLE SILLICON RECOVERY`** Failure to do so will make your mac unbootable!!

```bash
sudo fdisk /dev/nvme0n1
sudo reboot
```

## Optional: boot encryption (WIP)

<!--

copy the linux BOOT files

```
sudo dd if=/dev/nvme0n1p5 of=./mount/boot-backup bs=10M

```
umount boot

```
sudo umount /boot/efi
sudo umount /boot
```

create new LUKS on nvme0n1p5

```
sudo cryptsetup luksFormat --type luks1 /dev/nvme0n1p5
sudo cryptsetup open /dev/nvme0n1p5 BootDecrypted
sudo mkfs.ext4 /dev/mapper/BootDecrypted
```

copy original boot files

```
mkdir mount-boot
sudo mount /dev/mapper/BootDecrypted ./mount-boot
sudo losetup /dev/loop100 ./mount/boot-backup 
mkdir mount-boot-orig
sudo mount /dev/loop100 mount-boot-orig/
sudo rsync -aXv mount-boot-orig/ mount-boot/
sudo umount ./mount-boot ./mount-boot-orig 
sudo mount /dev/mapper/BootDecrypted /boot
sudo mount /dev/nvme0n1p4 /boot/efi
```

GRUB config enable cryptsetup

```
nvim /etc/default/grub
```

```
GRUB_CMDLINE_LINUX_DEFAULT="root=UUID=ROOTUUID cryptkey=rootfs:/root/bootkey cryptdevice=UUID=VGUUID:cryptvg loglevel=3 quiet intel_iommu=on iommu=pt pcie_ports=compat"
GRUB_ENABLE_CRYPTODISK=y
```

install grub

```
sudo grub2-install --target=arm64-efi --efi-directory=/boot/efi --bootloader-id=grub --force
sudo cp /boot/efi/EFI/grub/grubaa64.efi /boot/efi/EFI/fedora/grubaa64.crypt.efi
```

-->
