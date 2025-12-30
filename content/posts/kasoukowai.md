---
title: "仮想通貨こわいね..."
tags: []
date: 2022-07-01T19:44:19+09:00
---

polygonのRPCがジャックされたらしい...

https://twitter.com/NoxSayin/status/1542812886543634432

1.1.1.1が186.2.171.14を回答している

```
nslookup polygon-rpc.com 1.1.1.1
Server:		1.1.1.1
Address:	1.1.1.1#53

Non-authoritative answer:
Name:	polygon-rpc.com
Address: 186.2.171.14
```

これは以前にも仮想通貨関係の詐欺に使われているIPアドレスらしいね...

https://twitter.com/sniko_/status/1525247207301160960


IIJのDoHもそんな感じ

```
doh polygon-rpc.com https://public.dns.iij.jp/dns-query
[polygon-rpc.com]
TTL: 82003 seconds
A: 186.2.171.14
```

(https://github.com/curl/dohを使用)

8.8.8.8は無回答

```
nslookup polygon-rpc.com 8.8.8.8
Server:		8.8.8.8
Address:	8.8.8.8#53

Non-authoritative answer:
*** Can't find polygon-rpc.com: No answer
```



