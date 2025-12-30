---
title: "mac osのcurlがなんか変だぞ？"
tags: ["MacOS", "curl", "Apple", "DoH"]
date: 2023-01-22T11:21:30+09:00
---

Mac OSのcurl、（プライベートリレーを有効にしている場合）なにやらAppleの持ってるDNS-over-HTTPSみたいなものにアクセスしようとしている???

curlがインターネットにアクセスしようとするたびに、ここにリクエストが飛ばされている...

```
Standard query HTTPS mask.apple-dns.net
Standard query response HTTPS mask.apple-dns.net SOA ns-1462.awsdns-54.org
```

(というかHTTPSレコードってあるんだね...)

でもどうやらDoHでもないらしく？？かといって平文のDNSでもないらしい（HTTPSと書いてあるからそれはそうか）

しかし実のところ、標準のSSLですらないらしい??

<https://github.com/curl/doh>を使ってみると...

```
doh -v google.com https://mask.apple-dns.net
== Info: Found bundle for host mask.apple-dns.net: 0x600001454db0 [serially]
== Info: Server doesn't support multiplex (yet)
== Info:   Trying 2403:300:1363:4::9:443...
== Info: Hostname 'mask.apple-dns.net' was found in DNS cache
== Info:   Trying 2403:300:1363:4::9:443...
== Info: Connected to mask.apple-dns.net (2403:300:1363:4::9) port 443 (#0)
== Info: ALPN, offering h2
== Info: ALPN, offering http/1.1
== Info: successfully set certificate verify locations:
== Info:  CAfile: /etc/ssl/cert.pem
== Info:  CApath: none
== Info: (304) (OUT), TLS handshake, Client hello (1):
=> Send SSL data, 0000000323 bytes (0x00000143)
...
== Info: Connected to mask.apple-dns.net (2403:300:1363:4::9) port 443 (#1)
== Info: ALPN, offering h2
== Info: ALPN, offering http/1.1
== Info: successfully set certificate verify locations:
== Info:  CAfile: /etc/ssl/cert.pem
== Info:  CApath: none
== Info: (304) (OUT), TLS handshake, Client hello (1):
=> Send SSL data, 0000000323 bytes (0x00000143)
...
== Info: LibreSSL SSL_connect: SSL_ERROR_SYSCALL in connection to mask.apple-dns.net:443
== Info: Closing connection 0
probe for A failed: SSL connect error
== Info: LibreSSL SSL_connect: SSL_ERROR_SYSCALL in connection to mask.apple-dns.net:443
== Info: Closing connection 1
probe for AAAA failed: SSL connect error
```

証明書検証エラーとかだったらまだ許せるが、ハンドシェイク時点でこんな感じで接続エラーが出てくる！これはレアじゃないの？

curlに無理やり? AppleのDoHをねじ込むばかりでなく、接続先のサーバーも標準のDoHの実装ではないのだったら独自にAPIを用意しているということでまだいいが、SSLっぽいけど微妙に違う謎仕様という、さすがApple...

まあ全部のケースに対応してるとサーバーが重くなるからとかあるんだろうけどね。