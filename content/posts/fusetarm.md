---
title: "M2 Mac / aarch64 Mac fuse-t build scripts"
tags: []
date: 2025-06-30T19:27:03+09:00
---

this is a guide for building fuse applications along with fuse-t on ARM Mac OS.

A little bit different build processes will occur depending on whether you have installed pkg-config or not (pkg config can help you with some flags so you don’t have to follow the procedures below but might lead to complications such that library version mismatches or requiring both `.framework` and `.dylib` data)

Should work for both ways tho.

As with any other build config, just remember to add `-I/path/to/include` to cflags/cxxflags/cppflags, `-L/path/to/lib -F/Library/Frameworks -f framework_name` to LDFLAGS. If linker error still happens, you will need to pass linker options to CFLAGS/CXXFLAGS/CPPFLAGS also, such that, `-Wl,-F,/Library/Frameworks -Wl,-f,framework_name -Wl,-L,/path/to/lib`

Whenever you run a binary file, do it with `DYLD_FRAMEWORK_PATH=/Library/Frameworks DYLD_LIBRARY_PATH=/path/to/lib ` env variables. 

I haven’t tested yet, but fuse-t feels way slower than osxfuse, esp with ntfs-3g capped at 20MB/s or so..

# ext4fuse

Required Frameworks: fuse_t

build
```
#!/bin/bash
export CFLAGS=-I/Library/Frameworks/fuse_t.framework/Headers/
export LDFLAGS="-F/Library/Frameworks -framework fuse_t"
make
```

run
```
#!/bin/bash

cd $HOME/Desktop
DIRN="/Volumes/fuse$(date +%s%N)"
sudo mkdir "$DIRN"
sudo DYLD_FRAMEWORK_PATH=/Library/Frameworks ./ext4fuse/ext4fuse -o nfc "/dev/$1" "$DIRN"

# for osxfuse you can do something like "-o modules=iconv,from_code=UTF-8,to_code=UTF-8-MAC"

# if you want to use nfd, just delete "-o nfc" option
```

# fuse-ext2

Required Frameworks: fuse_t
Required dylibs: e2fsprogs (now can be retrieved via homebrew, without the e2fsprogs' very own fancy fuse script.)

Source patch
```
diff --git a/configure.ac b/configure.ac
index 36f0540..faf9575 100644
--- a/configure.ac
+++ b/configure.ac
@@ -105,10 +105,10 @@ AC_FUNC_CHOWN
 AC_FUNC_CLOSEDIR_VOID
 AC_FUNC_GETMNTENT
 AC_PROG_GCC_TRADITIONAL
-AC_FUNC_MALLOC
+# AC_FUNC_MALLOC
 AC_FUNC_MEMCMP
 AC_FUNC_MMAP
-AC_FUNC_REALLOC
+# AC_FUNC_REALLOC
 AC_FUNC_SELECT_ARGTYPES
 AC_FUNC_STAT
 AC_FUNC_UTIME_NULL
diff --git a/fuse-ext2/fuse-ext2.c b/fuse-ext2/fuse-ext2.c
index 249a389..f9e1188 100644
--- a/fuse-ext2/fuse-ext2.c
+++ b/fuse-ext2/fuse-ext2.c
@@ -298,7 +298,7 @@ static const struct fuse_operations ext2fs_ops = {
 	.release	= op_release,
 	.fsync          = op_fsync,
 	.setxattr       = NULL,
-	.getxattr       = op_getxattr,
+	.getxattr       = NULL,
 	.listxattr      = NULL,
 	.removexattr    = NULL,
 	.opendir        = op_open,
```

build
```
#!/bin/bash
export CFLAGS="-I/opt/homebrew/opt/e2fsprogs/include -I/Library/Frameworks/fuse_t.framework/Headers -F/Library/Frameworks -Wl,-F,/Library/Frameworks -Wl,-framework,fuse_t"
export LDFLAGS="-L/opt/homebrew/opt/e2fsprogs/lib -F/Library/Frameworks -framework fuse_t"
# [replace op_getxattr with NULL in fuse-ext2.c]
# [comment out AC_FUNC_MALLOC and AC_FUNC_REALLOC in configure.ac] a little bit dangerous but anyway. no one do horrible things like malloc(0) !
./autogen.sh
./configure --host=aarch64 --prefix="$PWD"
make
make install
```

run
```
#!/bin/bash
cd $HOME/Desktop
DIRN="/Volumes/fuse$(date +%s%N)"
sudo mkdir "$DIRN"
sudo DYLD_FRAMEWORK_PATH=/Library/Frameworks ./fuse-ext2/bin/fuse-ext2 -o ro,nfc "/dev/$1" "$DIRN"

```

# ntfs-3g

Required Frameworks: fuse_t

Plz use the forked version at <https://github.com/macos-fuse-t/ntfs-3g>

(or maybe you can make a custom fuse.pc file with modified versions?)

```
#!/bin/bash

git clone https://github.com/macos-fuse-t/ntfs-3g
sudo mkdir -p /usr/local/lib/pkgconfig/
brew install macos-fuse-t/homebrew-cask/fuse-t
brew install gcc make autoconf automake libtool libgcrypt pkg-config
export CFLAGS=-I/Library/Frameworks/fuse_t.framework/Headers/
export LDFLAGS="-F/Library/Frameworks -framework fuse_t"
./autogen.sh
./configure --prefix="$PWD" --host=aarch64-darwin --exec-prefix="$PWD"
make -j
make install

```

```
#!/bin/bash

sudo diskutil umount "$1"

cd $HOME/Desktop
DIRN="/Volumes/fuse$(date +%s%N)"
sudo mkdir "$DIRN"
sudo DYLD_FRAMEWORK_PATH=/Library/Frameworks DYLD_LIBRARY_PATH=/usr/local/lib ./ntfs-3g/bin/ntfs-3g -o nfc "/dev/$1" "$DIRN"
# for osxfuse you can do something like "-o modules=iconv,from_code=UTF-8,to_code=UTF-8-MAC"

```
