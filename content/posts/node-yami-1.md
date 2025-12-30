---
title: Nodeの闇其の壱
date: 2018-03-10T16:57:24+09:00
tags: ["javascript"]
---

Node.jsでは、os.platform()を呼ぶと、実行しているOSの種類がわかる。
この時、Linuxの場合は`'linux'`、Macの場合は`'darwin'`、そしてWindowsの場合は(64ビットでも)`'win32'`が返される。