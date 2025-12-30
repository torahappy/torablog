---
title: "通信内容の証明：ブラウザなどでサイトにアクセスする時の暗号鍵を取得する"
tags: ["通信", "証明", "openssl"]
date: 2020-02-23T22:40:17+09:00
---

Chrome, Firefox などで、  
環境変数`SSLKEYLOGFILE`
を設定しつつ起動すると、  
秘密鍵・復号暗号鍵をダンプできるぞ！

環境変数`SSLKEYLOGFILE`にはダンプ先のファイル名を指定する。例: `SSLKEYLOGFILE=~/keylog.txt`

多分opensslの機能。

HTTPS接続の場合は、Wiresharkなどで取ってきたパケットのダンプと合わせると、  
簡易的な通信内容の証明ができる！スクショだと偽造できるかもだけど、この方法なら安心！
