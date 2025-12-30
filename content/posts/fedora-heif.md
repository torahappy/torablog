---
title: "tech memo: fedora libheif"
tags: []
date: 2025-12-11T22:02:15+09:00
---

Unfortunately `libheif` in the default yum repository cannot handle most heic files due to missing libraries.

This is the guide on how to build libheif with required libraries.

## build custom libheif with various libraries

AOM
https://aomedia.googlesource.com/aom

```bash
mkdir dist
cd dist
cmake -DBUILD_SHARED_LIBS=ON -DCMAKE_INSTALL_PREFIX=$HOME/apps/aom-bin/ ..
make -j5
make install
```

libde265
https://github.com/strukturag/libde265

```bash
mkdir build
cd build
cmake -DCMAKE_INSTALL_PREFIX=$HOME/apps/libde265-bin ..
make -j5
make install
```


x265
https://github.com/videolan/x265

```bash
cd build/arm-linux
cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX=$HOME/apps/x265-bin/ ../../source
make -j5
make install
```

libheif
https://github.com/strukturag/libheif

```bash
sudo dnf remove libaom-devel
sudo dnf install zlib-ng-compat-devel libpng-devel libtiff-devel libjpeg-devel doxygen libopenjph-devel

mkdir build
cd build

PKG_CONFIG_PATH="$HOME/apps/aom-bin/lib64/pkgconfig:$HOME/apps/libde265-bin/lib64/pkgconfig/:$HOME/apps/x265-bin/lib/pkgconfig/" cmake DCMAKE_INSTALL_PREFIX="$HOME/apps/libheif-bin" ..

make -j5

make install

sudo dnf install libaom-devel
```

## Runtime options

At runtime, please put LD_LIBRARY_PATH as following:

```bash
LD_LIBRARY_PATH="$HOME/apps/libde265-bin/lib64/:$HOME/apps/x265-bin/lib/:$HOME/apps/libheif-bin/lib64"
```

(example)
```bash
sudo dnf install ristretto heif-pixbuf-loader libheif

export LD_LIBRARY_PATH="$HOME/apps/libde265-bin/lib64/:$HOME/apps/x265-bin/lib/:$HOME/apps/libheif-bin/lib64"

ristretto .
```

## Thumbnailer replacement

Unfortunately, you can't use thumbnailer at this time, because tumblerd uses `/usr/bin/heif-thumbnailer` with no `LD_LIBRARY_PATH` set.
`/usr/bin/heif-thumbnailer` may work well with above `LD_LIBRARY_PATH`, but it is safer to use the binary installed at `$HOME/apps/libheif-bin/bin`.
Thus, we need to replace the thumnailer itself, as a user-space configuration.
If you want to use system-space configuration but don't want to make mess with the yum/dnf, try `/usr/local/share/thumbnailers`.

To replace the default heif thumbnailer, create two files first.

`.local/share/thumbnailers/heif.thumbnailer` Contents:

```ini
[Thumbnailer Entry]
TryExec=/home/<your user name>/Desktop/scripts/heif-thumbnailer.sh
Exec=/home/<your user name>/Desktop/scripts/heif-thumbnailer.sh -s %s %i %o
MimeType=image/heif;image/avif;
```

`/home/<your user name>/Desktop/scripts/heif-thumbnailer.sh` Contents:

```bash
#!/bin/bash

export LD_LIBRARY_PATH="$HOME/apps/libde265-bin/lib64/:$HOME/apps/x265-bin/lib/:$HOME/apps/libheif-bin/lib64"

export PATH="$HOME/apps/libheif-bin/bin:$PATH"

heif-thumbnailer "$@"
```

DO NOT FORGET to append executable permissions to BOTH `heif-thumbnailer.sh` AND `heif.thumbnailer`.

```bash
chmod +x $HOME/Desktop/scripts/heif-thumbnailer.sh $HOME/.local/share/thumbnailers/heif.thumbnailer
```
