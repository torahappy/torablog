---
title: "Asahi Linux Wine 動かし方 メモ（2025 Dec 版）"
tags: []
date: 2025-12-03T07:21:00+09:00
---

## 最重要

ここでつまずいてたいへんだった...

Waylandじゃないと動きません!!!!!! Xは使わないこと!!!

## 登場人物たち

- BOX64 - なんか高速にx86_64をエミュレーションできるやつ  
  ライブラリもうまいことarm64とx86_64とをハイブリッドで使ってくれる
- fex-emu - また別のx86_64エミュレーター
- muvm - ページサイズ16k→4kにするやつ
- sommelier - muvmとWaylandをつなげる
- **wayland**のデスクトップマネージャ
- umu - protonをsteamにログインしなくても動かせるやつ
- proton - wineのめっちゃ改造しまくったバージョン

## 手順

### umuのインストール

ビルド自体は、x86_64環境で動かさない。実行時にエミュレーション環境に入る。

rustupインストール済みを想定。他にも色々基本的なビルドツールなど必要かも。makeとか

注意点: READMEのやり方のままだと、setuptoolsが足りないことがある。その場合のエラーメッセージがとても紛らわしいものが出てくるので注意すること。これ以外のdependencyエラーについては、(一応)わかりやすくは教えてくれる。

またインストールの際は、「現在使っているpython実行環境が存在するものと同じディレクトリ」を指定しなければ、必ずエラーになるので注意！！！

pythonが/usr/bin/pythonにあるのであれば必ず--prefix=/usrにしなければいけない。/umu/venv/bin/pythonであれば/umu/venvを指定する。

以上の条件をクリアーしてもなお、ビルド時に"setuptools-scm<9" "hatch-vcs<0.6.0"を入れろとか、実行時にXlibを入れろとか言われるので、そこもやっておく。

```bash
rustup toolchain install stable
sudo dnf install scdoc
mkdir umu
cd umu
python3 -m venv venv
. venv/bin/activate
git clone --recursive https://github.com/Open-Wine-Components/umu-launcher
cd umu-launcher
pip install build hatchling installer setuptools "setuptools-scm<9" "hatch-vcs<0.6.0" Xlib
./configure.sh --prefix="$(realpath ../venv)"
make
make install
```

### muvmとfex-emuを通してumuを動かす

/bin/steamをテキストエディタで開くと中でどういう処理になってるのかわかりやすい。参考までに。

あと`muvm FEXBash`内の実行環境は`#!`から始まる実行ファイルをうまく処理してくれないかも。。ここはわりといろんなバグの源になる可能性あり。

```
sudo dnf install steam sommelier fex-emu muvm

muvm FEXBash -it

# 上で開いたシェル内で実行
./venv/bin/python ./venv/bin/umu-run explorer
```

steamと打てばふつうにsteamも動く。

## 他のx86_64アプリについて

一般的に、Linux対応ソフトの場合は、

1. まずはBOX64
2. だめだったらmuvm
3. それでもだめだったらmuvm+fex

って感じかな？

ちなみに、BOX64を使う場合は、  
https://packages.fedoraproject.org/pkgs/gcc/libgcc/  
からx86_64版のlibgccを落としてきて、パスを通すといい。だいたいlibgccで怒られるけど、そこを超えたら行ける場合が多い。wineなど一部のアプリはページサイズ問題で落ちる。

でもwineが動くくらいだしmuvmも普通に優秀そう。muvm使う場合は、ゲームだったら、変に色んなとこからライブラリ引っ張ってくるよりも、大体のライブラリがすでに揃っているsteamrt/sniper上で動かすのもいいかも。vulkanとかシェーダーキャッシュとかも勝手に設定してくれるし。大量のGPL/LGPLバイナリが入ってるので、バリバリ他のアプリ動かすのに自由に使ってもいいよね。(そもそもログインユーザー限定で囲ってるほうがおかしい。)


BOX64: 普通に裸のシェルの状態で以下を実行すればOK. (qemu-user-x86などが入っているとBOX64と競合することがあるので、消しておくこと。)
```
$HOME/.local/share/umu/steamrt3/_v2-entry-point <app-path>
```

MUVM+FEX: FEXに入ってから同様に実行すればOK. Waylandになってることを確認！！
これで他のとこから取ってきたwineも、もろもろのライブラリ完備の状態で動かせるはず。
```
muvm FEXBash -it
$HOME/.local/share/umu/steamrt3/_v2-entry-point <app-path>
```

## その他メモ

wine ダウンロード元

- winehq builds  
  https://dl.winehq.org/wine-builds/fedora/42/x86_64/  
  steamrtとあわせるならfedoraじゃなくてdebian版がいいかも
- Kron4ek  
  https://github.com/Kron4ek/Wine-Builds/releases
- umu-proton  
  https://github.com/Open-Wine-Components/umu-proton
- proton-ge-custom  
  https://github.com/GloriousEggroll/proton-ge-custom

普通にumuが動いたので基本それでいいとは思うが、proton以外のwineを使う場合は、fedoraの公式からいろいろx86_64ライブラリーを引っ張ってきて別の場所に展開するプログラムを作る必要あるかも。。もしくはnixをつかう手もあり。もしくはsteamrtで全部入りを試す。

結論：色々やって試してみるしかない

box64→nixでライブラリの設定（どれをネイティブにするのかなど）を制御してみてもいいかも

カーネルのバージョンを落とす必要もあるかも。

カーネルのバージョンとwineのバージョン、wine のダウンロード元を色々変えてみて試してみる

16kページ関係の参考

Tristan Ross, Why 16k page size matters  
https://tristanxr.com/post/why-16k-page-size/

AAA gaming on Asahi Linux  
https://asahilinux.org/2024/10/aaa-gaming-on-asahi-linux/

Wine unable to load kernel32.dll  
https://github.com/FEX-Emu/FEX/issues/4261

BOX64は残念ながら使えない。  
arch linuxのBOX64では、ページサイズ16k vs 4k問題があるため、wineは動かない。。。。(Pi5ならカーネルを置き換えすることで行けるっぽい？ただ、Asahiはハードウェアの問題で多分4k対応してない。。)

qemuって使えるのかな、、  
QEMU with VirtIO GPU Vulkan Support : https://gist.github.com/peppergrayxyz/fdc9042760273d137dddd3e97034385f  
QEMU/Guest graphics acceleration : https://wiki.archlinux.org/title/QEMU/Guest_graphics_acceleration  
なども参考になるかも。

ページサイズが4kだったら、普通にbox64で動くんだけどね、、、悲しみ。

ARM64でwineをビルドする方法（あんま需要ないかな。。）  
https://www.reddit.com/r/AsahiLinux/comments/1k15eoi/native_arm64_wine_with_16k_page_support_incl_x64/?tl=ja

参考：  
https://github.com/asfdrwe/asahi-linux-translations/blob/main/PROGRESS202109.md から引用：  
IOMMU 4Kパッチ（レビュー中）: M1が特異なのは、16Kと4Kページのどちらかを使用するOSに対応しているにもかかわらず、実際には16Kシステム用に 設計されていることです。M1のDART IOMMUハードウェアは16Kページしか対応していません。このチップが4Kに対応しているのは主にmacOS上で Rosettaを動作させるためです。macOS自体は常に16Kページで動作しており、Rosettaアプリだけが4Kモードになっています。 Linuxでは、このようにページサイズを混在させることはできませんし、今後もできないでしょうから、難問が残ったままです。 16Kカーネルを実行していると、古いユーザースペース（主にAndroidやx86エミュレーション）との互換性が難しくなり、 またディストリビューションは通常16Kカーネルを出荷しません。一方で4Kカーネルを実行すると、DARTとの大きなミスマッチが発生します。 これは当初解決不可能な問題のように思えましたが、Sven氏がこの問題に取り組み、IOMMU のページサイズがカーネルのページサイズよりも 大きいハードウェアとうまく連携するようにLinux の IOMMU サポートレイヤーを対応させるパッチを作成しました。これは完全ではありません。 (この状況では根本的に対応できないことを行う)少数のコーナーケースドライバでは対応できないからです。しかし、これはうまく機能し、 4K カーネルを実現するために必要なすべてのことに対応します。

