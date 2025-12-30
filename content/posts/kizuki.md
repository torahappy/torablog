---
title: 気付き
date: 2019-10-23T00:12:00+09:00
tags: ["VirtualBox", "Wine"]
---

<https://forum.winehq.org/viewtopic.php?t=5216>

LINUX(MACでも？)でキー入力が荒ぶる時は、
/dev/input/js* (ジョイスティック入力)が
なぜかマウス入力に設定されていることがある！！！

vboxの設定でmouse integrationをオフにするか、

/dev/input/js0, /dev/input/js1を自動的に(10秒ごとにチェックするとかで)消去するか

しないといけない

