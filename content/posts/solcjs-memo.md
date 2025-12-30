---
title: solcをブラウザで使うときに詰まった点
tags: ["Ethereum", "solc", "javascript"]
date: 2018-03-26T15:25:25+09:00
---

SolidityのコンパイラはC++で書かれている。JSでは、Emscriptenを用いてsoljson.jsという名前のJavascript(asm.js)に変換した上で使っている。そのwrapperこそが、おなじみのsolcだ。ちなみに、今までのsoljson.jsは<https://github.com/ethereum/solc-bin>にある。

ではブラウザで使う場合はどうしよう？残念ながら[solcjs](https://github.com/ethereum/solc-js)はコンパイル済みのjsファイルを持っていないため、Webpackを使うことになるのだが、ここで一つ問題が発生する。soljson.jsをWebpackに突っ込むと、コンパイルが終わらない。Parcelなど使おうものならヒープ領域がパンクする。V8でOutOfMemoryなんて初めて見たぞ。これはこのファイルが7MB超と大きいのが理由だと思われる。しかしWebpackもそこらへん察してほしいな。というか、そもそもNode.js用にnode-gypではなくEmscriptenを使っているのもどうかと思う。あとwasmないの（>_<）

ということで愚直に`import solc from 'solc'`していてはコンパイルが終わらないので、soljson.jsをブラウザからscriptタグでそのまま読み込みつつ、ラッピングして自分でsolcオブジェクトを作るという謎の芸当をしなくてはいけない。ということで以下のコード。

solc.js
{{< highlight javascript >}}
import solc from 'solc/wrapper'
export default solc(require('module'))
{{< /highlight >}}

webpack.config.js
{{< highlight javascript >}}
module.exports = {
  .
  .
  .
  externals: {
    'module': 'Module'
  },
  .
  .
  .
}
{{< /highlight >}}

まず一行目で、solcの「ラッパー部分」を読み込んでいる。そして二行目で、読み込んだラッパーにscriptタグからsoljson.jsから生成されたModule変数を渡している。他のライブラリでrequire('module')としている箇所があるので、require('module')から一括してwindow.Moduleを取得するようにしている。で、これと似たようなことが[remixのソースコード](<https://github.com/ethereum/remix/blob/287aa1153f0bc5b5d0faf2d01d12ef5a8f6e10ca/remix-solidity/src/compiler/compiler.js#L238>)にも書かれている。うーん黒魔術。