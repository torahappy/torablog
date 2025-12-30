---
title: "Tech memo: ARCH LINUX ffmpeg build script with avisynthplus plugin"
tags: ["techmemo", "ArchLinux", "ffmpeg", "AviSynthPlus"]
date: 2025-01-22T23:43:55+09:00
---

## build avisynthplus

```bash
cd ~/
mkdir apps
cd apps
git clone --recursive https://github.com/AviSynth/AviSynthPlus
mkdir build
cd build
cmake -DCMAKE_INSTALL_PREFIX=$HOME/apps/AviSynthPlusBin ..
make -j10
make install
```

## run ffmpeg-build-script

Just running `build-ffmpeg` alone would probably result in various kinds of errors.

Here, we're doing following tweaks:

- Add `-fpermissive` to c and c++ flags to fix some build errors for the latest gcc/g++.
- Compile and install `cargo-c`  outside the `build-ffmpeg` build environment to avoid libssl linking issue.
- Add `/opt/cuda/include/` and `/opt/cuda/lib/` to the include and library path for building the ffmpeg binary. This is because the script is wrirtten for Ubuntu system, and cuda installation path is different between Arch and Ubuntu.
- Add `AviSynthPlusBin` path we've made in the previous section. 

build ffmpeg and other libs
```bash
# install cuda and build tools
sudo pacman -S cuda base-devel curl

# install rustup to local
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

. "$HOME/.cargo/env"
rustup update

# check rust version
rustc --version
cargo install cargo-cache

# preinstall cargo-c to work around with ssl 1.1 vs 3 linking issue
cargo uninstall cargo-c
cargo install cargo-c

# fix build error for some build dependencies
export CFLAGS="-fpermissive"
export CPPFLAGS="$CFLAGS"
export CXXFLAGS="$CXXFLAGS"

# clear cargo cache
cargo cache -a

# Optionally, run `git clean -dfx && git pull` to clear all the previous build files and fetch upstream code

# Here, add
#   -I/opt/cuda/include/
#   -I$HOME/apps/AviSynthPlusBin/include/
# to CFLAGS and add
#   -L/opt/cuda/lib/
#   -L$HOME/apps/AviSynthPlusBin/lib/ 
# to LDFLAGS and add
#   --enable-avisynth
# to the configure option.
vim ./build-ffmpeg

./build-ffmpeg --build --enable-gpl-and-non-free

echo "Version()" > test.avs

LD_LIBRARY_PATH="$HOME/apps/AviSynthPlusBin/lib/" ~/.local/bin/ffmpeg -f avisynth -i ./test.avs -profile:v high -vcodec h264_nvenc -pix_fmt yuv420p -cq:v 10 test.mov
```

build-ffmpeg patch script
```patch
diff --git a/build-ffmpeg b/build-ffmpeg
index b277711..d459dfb 100755
--- a/build-ffmpeg
+++ b/build-ffmpeg
@@ -9,12 +9,12 @@ SCRIPT_VERSION=1.54
 CWD=$(pwd)
 PACKAGES="$CWD/packages"
 WORKSPACE="$CWD/workspace"
-CFLAGS="-I$WORKSPACE/include -Wno-int-conversion"
-LDFLAGS="-L$WORKSPACE/lib"
+CFLAGS="-I/opt/cuda/include/ -I$HOME/apps/AviSynthPlusBin/include/ -I$WORKSPACE/include -Wno-int-conversion"
+LDFLAGS="-L/opt/cuda/lib/ -L$HOME/apps/AviSynthPlusBin/lib/ -L$WORKSPACE/lib"
 LDEXEFLAGS=""
 EXTRALIBS="-ldl -lpthread -lm -lz"
 MACOS_SILICON=false
-CONFIGURE_OPTIONS=()
+CONFIGURE_OPTIONS=("--enable-avisynth")
 NONFREE_AND_GPL=false
 DISABLE_LV2=false
 LATEST=false
```

## maybe docker is better way??

Maybe just running Dockerfile, with tweaks to also build AviSynthPlus, is better way, although imnot 100% sure that binaries made in Ubuntu works perfectly fine in Arch...