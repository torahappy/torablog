---
title: "Memo: using secure card on Linux Wine"
tags: ["techmemo"]
date: 2025-01-27T18:52:17+09:00
---


```bash
mkdir -p $HOME/apps/winscard-wine

cd $HOME/apps/winscard-wine

wget "https://dl.winehq.org/wine/source/6.x/wine-6.23.tar.xz"
tar xvf wine-6.23.tar.xz
mv ./wine-6.23 ./src-32
cd src-32
git init
git config user.name 'test'
git config user.email 'test@example.com'
git add .
git commit -m 'initial commit'
cd ..
cp -r ./src-32 ./src-64

function apply_patch () {
  cd $1
  patch -p1 < ../scard4wine-true-patch/all.patch
  cp ../scard4wine-true-patch/wine-compat56.h ./include/wine-compat.h
  cd ..
}

apply_patch src-32
apply_patch src-64
```

Here's the scard4wine patch data, modified from the original source (<https://sourceforge.net/projects/scard4wine/>) to work with wine 6.23.

[scard4wine-true-patch.tar](/bin/memo-winscard-wine/scard4wine-true-patch.tar)

Porting to the latest wine (like version 9 or 10) gonna be kinda tough works, beacuse now `.dll` and `.so` are completely separated (dll can't use unix functions and vice versa, we have to use some unix call bridge to communicate between them). Also install `pcsclite` and run `sudo systemctl start pcscd` .

```bash
export CFLAGS="-Wno-implicit-function-declaration"
export CPPFLAGS="$CFLAGS"
export CXXFLAGS="$CFLAGS"

cd src-64
./configure --enable-win64 --prefix=$PWD/../wine
make -j10
cd ..

cd src-32
PKG_CONFIG_PATH=/usr/lib32 ./configure --with-wine64=../src-64 --prefix=$PWD/../wine
make -j10
cd ..

cd src-64
make install
cd ..

cd src-32
make install
cd ..
```