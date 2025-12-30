---
title: "Tech memo: windows ffmpeg + ffms2 + avspmod + avisynthPlus build script"
tags: ["techmemo", "windows", "ffmpeg", "ffms2", "avspmod", "AviSynthPlus"]
date: 2025-01-26T03:50:36+09:00
---

All commands are PowerShell script. 

Let `C:\AvsWorkspace` be the build and install workspace.

## module description

- ffmpeg: A encoder/decoder for various audio and video files.
- ffms2: A plugin that allows to have ffmpeg files as source data in AviSynth.
- avspmod: An AviSynth script editor + previewer.
- avisynthPlus: A newer AviSynth implementation.

## install ffmpeg

First, install shared ffmpeg via chocolatey.

```
choco install ffmpeg-shared
```

As of 2025-01, the ffmpeg files are installed in `C:\ProgramData\chocolatey\lib\ffmpeg-shared\tools\ffmpeg-7.1-full_build-shared`. The version number may differ.

## install avisynthplus

Then, build avisynthPlus.

```powershell
cd C:\AvsWorkspace
git clone "https://github.com/AviSynth/AviSynthPlus" AviSynthPlus-build
cd AviSynthPlus-build
mkdir build
cd build
cmake ..
cmake --build . --config Release
cmake --install
cd ../../
cp -r "C:\Program Files (x86)\AviSynth+" AviSynthPlus-install
```

It seems inevitable that the build script writes some files to `C:\Program Files (x86)\AviSynth+` regardless of `CMAKE_INSTALL_PREFIX`.

## install zlib-ng

```powershell
cd C:\AvsWorkspace
git clone "https://github.com/zlib-ng/zlib-ng" zlib-ng-build
cd zlib-ng-build
mkdir build
cd build
cmake -DZLIB_COMPAT=ON "-DCMAKE_INSTALL_PREFIX=C:\AvsWorkspace\zlib-ng-install" ..
cmake --build . --config Release
cmake --install
cd ../../
```

## install ffms2

```powershell
cd C:\AvsWorkspace
git clone "https://github.com/FFMS/ffms2" ffms2-build
```

Rewrite vcxproj files as follows. Then open `build-msvc/ffms2.sln` and build the Release solution.

```diff
diff --git a/build-msvc/ffms2.vcxproj b/build-msvc/ffms2.vcxproj
index 293710a..ea470fb 100644
--- a/build-msvc/ffms2.vcxproj
+++ b/build-msvc/ffms2.vcxproj
@@ -144,12 +144,14 @@
       <IntrinsicFunctions>true</IntrinsicFunctions>
       <PreprocessorDefinitions>FFMS_EXPORTS;_CRT_SECURE_NO_WARNINGS;__STDC_CONSTANT_MACROS;NDEBUG;_WINDOWS;_USRDLL;FFMS2_EXPORTS;%(PreprocessorDefinitions)</PreprocessorDefinitions>
       <RuntimeLibrary>MultiThreaded</RuntimeLibrary>
+      <AdditionalIncludeDirectories>C:\ProgramData\chocolatey\lib\ffmpeg-shared\tools\ffmpeg-7.1-full_build-shared\include;C:\AvsWorkspace\zlib-ng-install\include;C:\AvsWorkspace\AviSynthPlus-install\include\avisynth</AdditionalIncludeDirectories>
     </ClCompile>
     <Link>
       <SubSystem>Windows</SubSystem>
       <EnableCOMDATFolding>true</EnableCOMDATFolding>
       <OptimizeReferences>true</OptimizeReferences>
-      <AdditionalDependencies>psapi.lib;uuid.lib;oleaut32.lib;shlwapi.lib;gdi32.lib;vfw32.lib;secur32.lib;ws2_32.lib;mfplat.lib;mfuuid.lib;strmiids.lib;ole32.lib;user32.lib;bcrypt.lib;crypt32.lib;%(AdditionalDependencies)</AdditionalDependencies>
+      <AdditionalDependencies>swresample.lib;swscale.lib;avformat.lib;avutil.lib;avcodec.lib;zlib.lib;AviSynth.lib;psapi.lib;uuid.lib;oleaut32.lib;shlwapi.lib;gdi32.lib;vfw32.lib;secur32.lib;ws2_32.lib;mfplat.lib;mfuuid.lib;strmiids.lib;ole32.lib;user32.lib;bcrypt.lib;crypt32.lib;%(AdditionalDependencies)</AdditionalDependencies>
+      <AdditionalLibraryDirectories>C:\AvsWorkspace\AviSynthPlus-install\lib;C:\AvsWorkspace\zlib-ng-install\lib;C:\ProgramData\chocolatey\lib\ffmpeg-shared\tools\ffmpeg-7.1-full_build-shared\lib;</AdditionalLibraryDirectories>
     </Link>
   </ItemDefinitionGroup>
   <ItemGroup>
diff --git a/build-msvc/ffmsindex.vcxproj b/build-msvc/ffmsindex.vcxproj
index b299b8c..d688a7e 100644
--- a/build-msvc/ffmsindex.vcxproj
+++ b/build-msvc/ffmsindex.vcxproj
@@ -149,11 +149,14 @@
       <IntrinsicFunctions>true</IntrinsicFunctions>
       <PreprocessorDefinitions>_CRT_SECURE_NO_WARNINGS;__STDC_CONSTANT_MACROS;NDEBUG;_CONSOLE;%(PreprocessorDefinitions)</PreprocessorDefinitions>
       <RuntimeLibrary>MultiThreaded</RuntimeLibrary>
+      <AdditionalIncludeDirectories>C:\ProgramData\chocolatey\lib\ffmpeg-shared\tools\ffmpeg-7.1-full_build-shared\include;C:\AvsWorkspace\zlib-ng-install\include;C:\AvsWorkspace\AviSynthPlus-install\include\avisynth</AdditionalIncludeDirectories>
     </ClCompile>
     <Link>
       <SubSystem>Console</SubSystem>
       <EnableCOMDATFolding>true</EnableCOMDATFolding>
       <OptimizeReferences>true</OptimizeReferences>
+      <AdditionalLibraryDirectories>C:\AvsWorkspace\AviSynthPlus-install\lib;C:\AvsWorkspace\zlib-ng-install\lib;C:\ProgramData\chocolatey\lib\ffmpeg-shared\tools\ffmpeg-7.1-full_build-shared\lib;</AdditionalLibraryDirectories>
+      <AdditionalDependencies>avutil.lib;AviSynth.lib;zlib.lib;$(CoreLibraryDependencies);%(AdditionalDependencies)</AdditionalDependencies>
     </Link>
   </ItemDefinitionGroup>
   <ItemGroup>
```

## Copy all the dll files to a directory.

Copy all the dll files (and some binaries) to a directory named `AviSynthPlusPlugins`.

```powershell
mkdir C:\AvsWorkspace\AviSynthPlusPlugins
cd "C:\AvsWorkspace\AviSynthPlusPlugins"
cp "C:\ProgramData\chocolatey\lib\ffmpeg-shared\tools\ffmpeg-7.1-full_build-shared\bin\*" .
cp "C:\AvsWorkspace\AviSynthPlus-install\lib\avisynth\*.dll" .
cp "C:\AvsWorkspace\AviSynthPlus-install\bin\AviSynth.dll" .
cp "C:\AvsWorkspace\zlib-ng-install\bin\zlib1.dll" .
cp "C:\AvsWorkspace\ffms2\build-msvc\bin\x64\Release\ffms2.dll" .
```

This fill be about a 200 MB folder filled with binaries.

## Finally, install avspmod

Download avspmod from `https://github.com/AvsPmod/AvsPmod/releases/download/v2.5.1/AvsPmod_v2.5.1.zip` and unpack it somewhere.

Create a registry key `HKEY_LOCAL_MACHINE\SOFTWARE\AviSynth`. Then, create a string named `plugindir2_5` within it, and set the value to `C:\AvsWorkspace\AviSynthPlusPlugins`.

Copy `C:\AvsWorkspace\AviSynthPlus-install\bin\AviSynth.dll` to the AvsPmod folder.

Done!

## Also, you can pass avs files to ffmpeg.exe

Also, you can pass avs files to `ffmpeg.exe` in the `C:\AvsWorkspace\AviSynthPlusPlugins`, just like other video files.

Example: `C:\AvsWorkspace\AviSynthPlusPlugins\ffmpeg.exe -i MyVideo.avs -acodec aac -vcodec nvenc_h264 -profile:v high -g 150 -b:v 0 -cq 20 -bf 0 MyVideo.mov`

FFmpeg loads `AviSynth.dll` in the `AviSynthPlusPlugins` directory and then AviSynthPlus loads other plugins including `ffms2.dll` and FFMS2 uses loaded ffmpeg dlls.

## media-autobuild_suite

Instead of using chocolatey's shared ffmpeg, you can customize ffmpeg build via `media-autobuild_suite`.

`git clone https://github.com/m-ab-s/media-autobuild_suite` and double-click `media-autobuild_suite.bat`.

It is advised that the build environment should be placed right under `C:\` with shorter folder name, such as `C:\ffmp`. Then all the binaries will be installed on `C:\ffmp\local64`. The shared FFmpeg build, which we need for avisynth, is located on `C:\ffmp\local64\bin-video\ffmpegSHARED\bin`. Just copy the binaries to the avisynth plugins directory. Please also copy `C:\ffmp\local64\bin-video\libopenh264-7.dll` to that folder.

Here's example configuration files. Having too much dependencies will result in build error (too long arguments), so I omitted older/unpopular codecs from the list. 

ffmpeg_options.txt
```bash
# Lines starting with this character are ignored
# To override some options specifically for the shared build, create a ffmpeg_options_shared.txt file.

# Basic built-in options, can be removed if you delete "--disable-autodetect"
--disable-autodetect
--enable-amf
--enable-bzlib
--enable-cuda
--enable-cuvid
--enable-d3d12va
--enable-d3d11va
--enable-dxva2
--enable-iconv
--enable-lzma
--enable-nvenc
--enable-schannel
--enable-zlib
--enable-sdl2
--enable-ffnvcodec
--enable-nvdec
--enable-cuda-llvm

# Common options
--enable-gmp
--enable-libmp3lame
--enable-libopus
--enable-libvorbis
--enable-libvpx
--enable-libx264
--enable-libx265
--enable-libdav1d
--enable-libaom
--disable-debug
--enable-libfdk-aac

# Zeranoe
--enable-fontconfig
--enable-gnutls
--enable-libass
--enable-libbluray
--enable-libfreetype
--enable-libharfbuzz
--enable-libvpl
--enable-libmysofa
# --enable-libopencore-amrnb
# --enable-libopencore-amrwb
--enable-libopenjpeg
--enable-libsnappy
--enable-libsoxr
# --enable-libspeex
# --enable-libtheora
--enable-libtwolame
--enable-libvidstab
# --enable-libvo-amrwbenc
--enable-libwebp
--enable-libxml2
--enable-libzimg
--enable-libshine
--enable-gpl
--enable-openssl
--enable-libtls
--enable-avisynth
# --enable-mbedtls
# --enable-libxvid
--enable-libopenmpt
--enable-version3
--enable-librav1e
--enable-libsrt
# --enable-libgsm
--enable-libvmaf
--enable-libsvtav1

# Full
--enable-chromaprint
--enable-decklink
--enable-frei0r
--enable-libaribb24
--enable-libbs2b
--enable-libcaca
--enable-libcdio
--enable-libflite
--enable-libfribidi
--enable-libgme
# --enable-libilbc
--enable-libsvthevc
--enable-libsvtvp9
--enable-libkvazaar
--enable-libmodplug
# --enable-librist
--enable-librtmp
--enable-librubberband
#--enable-libssh
--enable-libtesseract
# --enable-libxavs
--enable-libzmq
--enable-libzvbi
--enable-openal
# --enable-libcodec2
--enable-ladspa
#--enable-vapoursynth
#--enable-liblensfun
--enable-libglslang
--enable-vulkan
# --enable-libdavs2
# --enable-libxavs2
# --enable-libuavs3d
--enable-libplacebo
--enable-libjxl
--enable-libvvenc
--enable-libvvdec
# --enable-liblc3

# Full plus options that add shared dependencies
--enable-opencl
--enable-opengl
--enable-cuda-nvcc
--enable-libnpp
--enable-libopenh264
```

media-autobuild_suite.ini
```ini
[compiler list] 
arch=3
license2=1
standalone=1
av1an=3
vpx2=1
aom=1
rav1e=1
dav1d=1
libavif=1
jpegxl=1
x2643=1
x2652=1
other265=1
svthevc=1
xvc=2
vvc=1
uvg266=1
vvenc=1
vvdec=1
svtav1=1
svtvp9=1
flac=1
fdkaac=1
faac=1
exhale=1
mediainfo=1
soxB=1
ffmpegB2=4
ffmpegPath=https://git.ffmpeg.org/ffmpeg.git
ffmpegUpdate=1
ffmpegChoice=1
mp4box=1
rtmpdump=1
mplayer2=2
mpv=1
vlc=2
bmx=1
curl=1
ffmbc=2
cyanrip2=2
ripgrep=1
jq=1
jo=1
dssim=1
avs2=1
dovitool=1
hdr10plustool=1
CC=2
cores=10
deleteSource=1
strip=1
pack=2
logging=1
updateSuite=1
timeStamp=1
ccache=1
noMintty=2
pkgUpdateTime=86400
```

## about linux...

In linux, the build process is much more easier (or maybe harder), however we don't have the great editor avspmod...

See [this link](./arch-linux-ffmpeg-build-script-avs.md) for building ffmpeg in Arch Linux.

(FYI in arch linux installation with a pacman's ffmpeg + avisynthplus caused a strange segfault issue so i had to build ffmpeg + avisynthplus)

maybe vapoursynth is also a good option to consider...

