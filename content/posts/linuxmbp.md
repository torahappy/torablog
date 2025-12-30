---
title: "Macbook Pro 2018でArch Linuxを動かすことに成功！！！"
date: 2020-03-16T00:58:00+09:00
---

Macbook Pro 2018でArch Linuxを動かすことに成功！！！

大変だった。

Macbook Proの中にインストールもできるけど、今回はUSBにインストールした。

## 方法

やる際は自己責任でお願いします。。。。

### 仮想マシン準備編 

1. arch linux の iso をダウンロード
1. VirtualBoxをインストール
1. VirtualBox extension pack をインストール
1. VirtualBoxで64bit仮想マシンを作成
1. USBメモリをMacbook Proに挿す
1. VirtualBoxの仮想マシン設定で、ポート -> USBにいき、USB 3.0コントローラを有効にする。さらに、USBデバイスフィルターに先ほど刺したUSBメモリを登録。システム -> EFIを有効化をオンにする。
1. 仮想マシンを起動。arch linux の iso を設定する。

### パーティション編(arch-chrootまで編)

1. 起動したら、/etc/pacman.confに以下を追加(nanoなどで編集)

   ```
   [mbp]
   Server = https://packages.aunali1.com/archlinux/$repo/$arch
   SigLevel = TrustAll
   ```

   以降は、<https://wiki.archlinux.org/index.php/Installation_guide>に従って設定する。キーボード・タイムゾーン・言語・パスワードの設定は省く。

1. `fdisk /dev/sdb`でUSBのパーティションを設定する。GUID Partition Tableで初期化し、まず最初に300MiB ~ 500MiBくらいのパーティション(`/dev/sdb1`)を作成し、次に余った領域でパーティション(`/dev/sdb2`)を作成する。最初のパーティションのtypeはEFI Systemに設定する。
1. `mkfs.fat -F 32 /dev/sdb1`
1. `mkfs.ext4 /dev/sdb2`
1. `mount /dev/sdb2 /mnt`
1. `mkdir /mnt/efi`
1. `mount /dev/sdb1 /mnt/efi`
1. `pacstrap /mnt base linux-mbp linux-firmware linux-mbp-headers`
1. `genfstab -U /mnt >> /mnt/etc/fstab`
1. `arch-chroot /mnt`

### ブートローダー編

1. `mkinitcpio -p linux-mbp`
1. `pacman -S grub efibootmgr`
1. `grub-install --target=x86_64-efi --efi-directory=efi --bootloader-id=GRUB`
1. `/efi/GRUB/grubx64.efi` を `/efi/boot/bootx64.efi`にコピー
1. `grub-mkconfig -o /boot/grub/grub.cfg` /efi/boot/grub.cfgと/efi/GRUB/grub.cfgにも生成した方がいいかも。
1. `pacman -S networkmanager iwd`
1. `systemctl enable NetworkManager.service`
1. `exit`
1. `reboot`で再起動。再起動後、USBメモリからブートするはず。

### ドライバ祭り編

1. `pacman -S git make nano gcc`
1. `git clone --branch mbp15 https://github.com/roadrunner2/macbook12-spi-driver`
1. `cd macbook12-spi-driver`
1. `make`
1. `mv apple-ibridge.ko apple-ib-tb.ko apple-ib-als.ko /lib/modules/*/`
1. `cd ../`
1. `git clone https://github.com/MCMrARM/mbp2018-bridge-drv.git`
1. `cd mbp2018-bridge-drv`
1. `make`
1. `mv bce.ko /lib/modules/*/`
1. `cd ../`
1. `depmod`
1. `/etc/modules-load.d/mbp.conf`を編集し、以下を追加。
   ```
   bce
   apple-ib-tb
   apple-ib-als
   ```
1. `reboot`で再起動。
1. 再起動後、`lsmod`を実行し、上の3つがリストに入っているか確認する。

### Wi-fi編

1. Mac OS側で`ioreg -l | grep C-4364`を実行。出てくる`.trx`, `.clmb`, `.txt`ファイルのパスをメモ。
1. Max OSの`/usr/share/firmware/wifi`下にそれらのファイルがあるので、3つを圧縮してどこかにコピー。(ここでは`~/wifi`とする。)
1. 1. `.trx`ファイルを`brcmfmac4364-pcie.bin`
   1. `.clmb`ファイルを`brcmfmac4364-pcie.clm_blob`
   1. `.txt`ファイルを`brcmfmac4364-pcie.Apple Inc.-[機種ID].txt`

   にリネーム。`[機種ID]`は「このMacについて -> システムレポート」で出てくる`MacBookPro15,1`などを入れる。
1. 先ほどの3ファイルをtarで圧縮。(ここでは`~/wifi/wifi.tar`とする。)
1. `browser-sync start -s ~/wifi` などで`~/wifi`下のファイルをLinuxと共有できるようにする。
1. Linuxに戻り、`curl -O http://[サーバのIPアドレス:ポート]/wifi.tar; tar xvf wifi.tar`でtarを解凍。
1. 3ファイルを`/lib/firmware/brcm/`に移動。
1. `/etc/NetworkManager/NetworkManager.conf`に以下を追加。
   ```
   [device]
   wifi.backend=iwd
   ```
1. `shutdown now`

### 感動の起動

1. MacBook Proを終了し、Optionを押しながら起動。
1. (多分) キーボードやTouch Barがちゃんと動いているはず。。。

## 参考

以下のリンクを参考にした。

<https://gist.github.com/TRPB/437f663b545d23cc8a2073253c774be3>

<https://ichelm.hatenadiary.org/entries/2014/12/09>

<http://www.silex.jp/blog/wireless/2015/08/linux.html>

## 追記
今回は3つのドライバ・カーネルパッチを導入した。主な機能は、

- linux-mbp : SSDとの通信
- macbook12-spi-driver : キーボード・Touch Barとの通信
- mbp2018-bridge-drv : T2チップとの通信

かな？

それぞれ機能がかぶさっている感じがするので、もしかすると全部を導入しなくてもいいかも。研究よろしく。

VirtualBoxでbootable USBを作るのをはやらせたい。wifi関係とかUSBキーボード用意するとかめんどくさいことをしなくて済む！！！

## 追記2

MacbookPro本体のSSDにインストールできたので報告。やり方はこんなかんじ。

1. SSDのパーティションをMacのディスクユーティリティで編集し、100GBの領域を作り、ExFATなどでフォーマット。
1. 最初の方法で作ったUSBでArchを起動。`pacman -S arch-install-scripts`でpacstrapを入れる。
1. `fdisk /dev/nvme0n1`で先ほど作ったパーティションを確認。OSXのパーティションを間違って選ばないように注意！（このときは`/dev/nvme0n1p3`だった。）
1. `mkfs.ext4 /dev/nvme0n1p3`で先ほどのパーティションをext4でフォーマットする。
1. `mount /dev/nvme0n1p3 /mnt`
1. `mkdir /mnt/boot`
1. `mount /dev/nvme0n1p1 /mnt/boot`
1. `pacstrap /mnt base linux-mbp linux-firmware linux-mbp-headers`
1. `genfstab -U /mnt >> /mnt/etc/fstab`
1. `arch-chroot /mnt`
1. `pacman -S networkmanager iwd`
1. `systemctl enable NetworkManager.service`
1. 「ドライバ祭り編」「Wi-fi編」と同様にドライバをインストールし、Wi-fiのファームウェアをコピー。
1. `pacman -S grub efibootmgr`
1. `grub-install --target=x86_64-efi --efi-directory=efi --bootloader-id=GRUB --no-bootsector --no-nvram`（注意！`--no-bootsector --no-nvram`をつけないとカーネルパニック？する。）
1. `grub-mkconfig -o /boot/grub/grub.cfg`
1. `mkinitcpio -p linux-mbp`
1. `exit`
1. `reboot`で再起動。

あとはrefind <https://www.rodsbooks.com/refind/installing.html> を入れるなどしてgrubを起動できるようにすればOK!

直接`sudo diskutil mount /dev/disk0s1` -> `sudo bless --mount /Volumes/EFI --setBoot --file /Volumes/EFI/efi/GRUB/grubx64.efi --shortform`でarchのefiをblessする方法もある。

## 追記3

Arch起動後にUSBで外付けドライブを接続した時に、なぜか認識されない(`/dev/sd*`がnot foundになる)場合がある。そういう時は、なぜか`lspci`コマンドを走らせると認識されるようになる！謎。起動する前から接続していた場合は大丈夫だった。USBマウスなどはそういうことをしなくても自動的につながった。
