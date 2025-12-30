---
title: BitcoinのAddressフォーマットis闇鍋
tags: ["Bitcoin"]
date: 2018-11-15T19:26:00+09:00
---

- base64形式:
  - p2pkh (20 bytes key hash) - `1` から始まる (testnet: `n`/`m`)
  - p2sh (20 bytes script hash) - `3` から始まる (testnet: `2`)
    - p2wpkh-nested-in-p2sh (`script hash = hash160(0014<20 byte key hash>)`)

- bech32形式 - `bc1` から始まる (testnet: `tb1`):
  - p2wpkh (20 bytes key hash)
  - p2wsh (32 bytes script hash)

同じアドレスを5種類の形で書き表すことができるなんて素敵！！！