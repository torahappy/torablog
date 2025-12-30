---
title: "MacOS wine build and install 2025 nov"
tags: []
date: 2025-11-18T22:04:40+09:00
---

M2 macOS wine build and install, 2025 November version (with DXMT / DXVK / D3DMetal)

## M2 macOS wine install 2025/11

### Pre-Built binaries

Repo: https://github.com/Gcenx/macOS_Wine_builds  

Run `brew install --cask --no-quarantine wine@devel` via homebrew

### Custom build

1.  Make sure `/etc/path.d` contains no files other than `10-cryptex`. Check for the user `.zprofile` / `.zshrc` and remove homebrew references. Reload the shell and make sure there is no `/opt/homebrew/bin` in `$PATH`. (Alternatively you can move `/opt/homebrew` to `/opt/off/homebrew`).
2.  Install MacPorts (see https://github.com/Gcenx/crossover-wine-ci/blob/blah/.github/workflows/bootstrap.sh )

```bash
MACPORTS_VERSION="2.11.6"

OS_MAJOR=$(uname -r | cut -f 1 -d .)
OS_ARCH=$(uname -m)
case "$OS_ARCH" in
    i586|i686|x86_64)
        OS_ARCH=i386
        ;;
    arm64)
        OS_ARCH=arm
        ;;
esac

MACPORTS_FILENAME=MacPorts-${MACPORTS_VERSION}-${OS_MAJOR}.tar.bz2

/usr/bin/curl -fsSLO "https://github.com/macports/macports-ci-files/releases/download/v${MACPORTS_VERSION}/${MACPORTS_FILENAME}"

# deactivate spot light
sudo mdutil -a -i off

sudo tar -xpf "${MACPORTS_FILENAME}" -C /

rm -f "${MACPORTS_FILENAME}"

source /opt/local/share/macports/setupenv.bash

# might be required to prevent the building process hanging up.
echo "ui_interactive no" | sudo tee -a /opt/local/etc/macports/macports.conf >/dev/null

# we might want to deactivate mirrors (because some mirrors are slow), but the main CDN server might also be slow. Alternatively, you can deactivate certain domains of mirrors only.
echo "host_blacklist *.distfiles.macports.org *.packages.macports.org" | sudo tee -a /opt/local/etc/macports/macports.conf >/dev/null

# we skip the archive_site_local settings in the original script

# required to prevent building arm binaries
echo "build_arch x86_64" | sudo tee -a /opt/local/etc/macports/macports.conf >/dev/null

# load sources
echo "file:///opt/macports-mingw-overrides" | sudo tee /opt/local/etc/macports/sources.conf >/dev/null
echo "file:///opt/macports-wine" | sudo tee -a /opt/local/etc/macports/sources.conf >/dev/null
echo "rsync://rsync.macports.org/macports/release/tarballs/ports.tar [default]" | sudo tee -a /opt/local/etc/macports/sources.conf >/dev/null

# initialize MacPort
sudo /opt/local/libexec/macports/postflight/postflight

# clone cusrom sources
cd /opt
sudo git clone --depth=1 https://github.com/Gcenx/macports-mingw-overrides.git
sudo git clone --depth=1 https://github.com/Gcenx/macports-wine.git

# sync MacPort
sudo port sync >/dev/null
sudo port sync -v

# build and install wine-devel
sudo port install wine-devel +gstreamer +kerberos
```

You can modify `/opt/macports-wine/emulators/wine-devel` to introduce a custom wine patch

## DXVK

### pre-built binaries

Download the custom DXVK for macOS here: https://github.com/Gcenx/DXVK-macOS/releases/tag/v1.10.3-20230507-repack

As usual, place x64 files to `<WINEPREFIX>/drive_c/windows/system32`  
And, place x32 files to `<WINEPREFIX>/drive_c/windows/syswow64`  
Note that the 64 bit binaries go to system *32* and, 32 bit binaries go to syswow *64*

Then, run `winecfg` and add `d3d10core`, `d3d11` to the dll list as `native then builtin`.  

You can activate/deactivate DXVK by appending/deleting these items from the list.
Or, if you specify EXE name on the first pane beforehand, different settings will be used per EXE files.
Or, you can just set the environment variable like `WINEDLLOVERRIDES="d3d10core,d3d11=n,b"` to activate DXVK (the method used by Proton!).

## DXMT

As far as i tested, DXMT does not work for custom-built Wine binaries (`/opt/local/lib/wine`) due to some symbols-related errors, only works for the homebrew Wine binaries. Adjustment of the build configuration is probably required for the custom Wine build. To patch the source, just put the patch files to `/opt/macports-wine/emulators/wine-devel` and change `Portfile`. (See https://github.com/3Shain/dxmt/wiki/Device-System-Runtime-Specifications for detail ; I have put `__attribute__((visibility("default")))` to several `winemac.so` functions, but that was't successful :( )

### pre-built binaries

Sadly, dlls placed on `<WINEPREFIX>/drive_c/windows/` and those placed on `/opt/local/lib/wine` or `/Applications/Wine Devel.app/Contents/Resources/wine/lib/wine` have different effects. Pre-built DXVK cannot be placed in `/opt/local/lib/wine` or `/Applications/Wine Devel.app/Contents/Resources/wine/lib/wine` and pre-built DXMT cannot be placed in `<WINEPREFIX>/drive_c/windows/`.

If you use pre-built DXMT, you have to place them in `/Applications/Wine Devel.app/Contents/Resources/wine/lib/wine`. You have to place 64-bit dlls to `x86_64-windows`, 32-bit dlls to `i386-windows` and 64-bit so files to `x86_64-unix`. See https://github.com/3Shain/dxmt/wiki/DXMT-Installation-Guide-for-Geeks .

Download it from here: https://github.com/3Shain/dxmt/releases/tag/v0.71

## D3DMetal / game-porting-toolkit

### custom build

Run `port install game-porting-toolkit`

If you are asked downloading D3DMetal, download from here https://developer.apple.com/jp/games/game-porting-toolkit/ (the URL provided by the shell won't work)

You can find another instance of wine in `/opt/local`. just search for the `game-porting-toolkit` directory.

As far as i tested, i couldn't use the D3DMetal files on `/opt/local/lib/wine` or `Wine Devel.app`.
