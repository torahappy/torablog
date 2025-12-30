---
title: infuraを使うときに詰まった点
tags: ["Ethereum", "INFURA", "javascript"]
date: 2018-02-16T15:25:25+09:00
---

INFURAではfilterが使えない。 
例えば、下記のコードは動かない。
{{<highlight javascript>}}
var web3 = new Web3(new Web3.providers.HttpProvider("https://mainnet.infura.io/"));
var filter = web3.eth.filter('pending');
filter.watch(console.log); // returns invalid json response error
{{</highlight>}}

コントラクトのEventを監視することもできない。
{{<highlight javascript>}}
var event = myContractInstance.MyEvent()
event.watch(console.log); // nothing will happen
{{</highlight>}}

試しに、json rpcを投げてみる。
{{<highlight bash>}}
$ curl https://mainnet.infura.io/ -X POST --data '{"jsonrpc":"2.0","method":"eth_newPendingTransactionFilter","params":[],"id":73}'
{{</highlight>}}
何も返ってこない。ステータスコードは405(Method Not Allowed)だった。

API Keyをつけても変わらなかった。

INFURAを使っているクライアントの一つ、MetaMaskでは、どうやら内部でフィルターしているらしい。
<https://ethereum.stackexchange.com/questions/34698/how-does-metamask-watch-filter-events>

というわけで、やっぱり自分でノード立てたほうがいい。