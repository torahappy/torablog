---
title: "ｇｈｃｊｓついに復活！！GHC 9.10 with JS backend でreflexも動くよ！！"
tags: ["techmemo", "Haskell"]
date: 2024-11-16T16:25:55+09:00
---

ちょーひさしぶりに技術メモ。

ついにGHC本体にJS backendが実装！！

ただしstackもnixもあまり対応していないので、cabalだけですべて行う必要あり。  
stackに関してはまだ調整すればなんとかなるかも？？  
（もうcabalだけでよくないか？stackもnixもいるのかなあ、、）

WASM  
```
# basic setup
ghcup config add-release-channel https://raw.githubusercontent.com/haskell/ghcup-metadata/develop/ghcup-cross-0.0.8.yaml
ghcup install cabal 3.12
git clone https://gitlab.haskell.org/ghc/ghc-wasm-meta.git
cd ghc-wasm-meta/
git checkout 18666510d272edaebdd27827b24bea563ebb6fb9
export SKIP_GHC=yes
./setup.sh
source ~/.ghc-wasm/env
ghcup install ghc --set wasm32-wasi-9.8.1 -- --host=x86_64-linux --with-intree-gmp --with-system-libffi


# build
echo 'main = putStrLn "hello world"' > hello.hs
wasm32-wasi-ghc hello.hs -o hello.wasm
wasmtime ./hello.wasm
```

GHCJS  
```
# load channel
ghcup config add-release-channel https://raw.githubusercontent.com/haskell/ghcup-metadata/develop/ghcup-cross-0.0.8.yaml
ghcup install cabal 3.12

# install emsdk
git clone https://github.com/emscripten-core/emsdk.git
cd emsdk
source ./emsdk_env.sh # add to .bashrc 

# specific set of versions for 9.10.0
emsdk install 3.1.57
emsdk activate 3.1.57
emconfigure ghcup install ghc --set javascript-unknown-ghcjs-9.10.0

# let's build
echo 'main = putStrLn "hello world"' > hello.hs
javascript-unknown-ghcjs-ghc -fforce-recomp hello.hs
./hello

### DO NOT use emconfigure with stack/cabal

# for cabal, works out of the box
cabal build --with-ghc javascript-unknown-ghcjs-ghc-9.10 --with-ghc-pkg javascript-unknown-ghcjs-ghc-pkg-9.10

# for stack, not working very well. also there is no stackage for 9.10
# Add following:
###############################################
configure-options:
  $everything:
    - --with-ld
    - emcc
    - --with-gcc
    - emcc
    - --with-ghc
    - javascript-unknown-ghcjs-ghc-9.10.0
    - --with-ghc-pkg
    - javascript-unknown-ghcjs-ghc-pkg-9.10.0
###############################################

stack build
```

cabalはうまく行くけどstackがだめな感じだね、、、、、なんかversion mismatchでこける、、cabal.projectとかであれこれやれば大丈夫なのかなあ。。てか、nixでもstackでもなく、自作スクリプトとかでstackとかnix/flakeのデータベースをもとにcabal.projectを自動生成すれば万事解決な気がしてきた。ビルド失敗するたびに、cabal.projectにallow-newer書いていくとか。

ghcjsならふつうにTHも動くし申し分ない  
たぶんjsaddleとかも大丈夫っぽい  
早く対応してくれるといいのだが、、、

**cabal build するときにemconfigureは使わないこと**。アウトプットに謎の#!が挿入されだめになってしまう。

パッケージマネージャ作るんだったら、自分で頑張ってnix書くしか無いのかなあ、、、  
環境作って、ghcを全部置き換えるだけだから簡単かなあ？？よくわからないけど、、でもあれかー、cabalがやってたこと全部nixでやり直さないといけないからむずそう、、  
nixでやるにしても、cabalを直接叩いたほうがいいのかなあ、、それってべつにnixじゃなくてよくね？ってなるけど、、いうてpackage.cabal有効活用すればreproductivityも担保できるのでは。

### nixのあれこれ

まったくうまく動かない謎のやつ
```
nix-build "<nixpkgs>" -A pkgsCross.ghcjs.haskell.packages.ghc96.reflex-dom
```

キャッシュ用テンプレ
```
nix-build --option extra-binary-caches https://nixcache.reflex-frp.org --option binary-cache-public-keys "cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY= ryantrinkle.com-1:JJiAKaRv9mWgpVAz8dwewnZe0AzzEAzPkagE9SP5NWI="
```

なんかリストするやつ
```
nix-env -f "<nixpkgs>" -qaP -A haskell.compiler
```

ghc本体をビルドする環境をつくるためのコマンド。ただしプロジェクトのビルド機能はないっぽい
```
nix develop git+https://gitlab.haskell.org/ghc/ghc.nix#js-cross --extra-experimental-features nix-command --extra-experimental-features flakes
```

なんかghcjsのビルド環境自体は、まだnixファイルすら作られてないかんじだね、、  
全部自作するしか無いのか、、

wasmのほうがghcjs よりも簡単そう。ghc-wasm-metaが用意されてるので。コンパイラじゃなくてアプリのほうをビルドできるようのnixファイルが用意されてる（ただwasmだとTHが使えないらしい？ので、、、、あとhaskellでもrustでも経験上、wasmは現段階だとなぜかロード時間がめっちゃ遅くて（でもEasyRPGとか、emscriptenのプロジェクトは結構速い）、ghcjsのほうがなんとなく速そう。）
```
nix shell gitlab:haskell-wasm/ghc-wasm-meta?host=gitlab.haskell.org --extra-experimental-features nix-command --extra-experimental-features flakes

cabal build --with-ghc=wasm32-wasi-ghc  --with-ghc-pkg=wasm32-wasi-ghc-pkg
```
wasmはロードにめっちゃ時間かかる印象しか無い、、、  
emscriptenも内部でwasm使ってるはずなんだけどね、  
emscriptenが優秀なのかV8が優秀なのか、GHC JS backendが優秀なのか、けっこう爆速だよね

やっぱnix じゃなくてstackが一番かなあ、、、やろうとおもえばけっこう変えることは可能だし

### 参考

<https://nixos.wiki/wiki/Haskell>


### やってるひとたち

Wasm build: <https://github.com/tweag/ghc-wasm-reflex-examples>（ちなみにここのcabal.projectを使ってwasmでなくghcjs版も普通にビルドできた。ロードも一瞬だしghcjsのほうが良さそう） 

Reflex使いたい人は、こんな感じのやつを作ればok.

cabal.project
```yaml
packages: .

with-compiler: javascript-unknown-ghcjs-ghc-9.10
with-hc-pkg: javascript-unknown-ghcjs-ghc-pkg-9.10

source-repository-package
  type: git
  location: https://github.com/amesgen/reflex-dom
  tag: e43e0525d643f656a0a5b0f10e13e2a04712cd4e
  subdir: reflex-dom-core reflex-dom

```

tmp.cabal
```yaml
cabal-version:      3.0
-- The cabal-version field refers to the version of the .cabal specification,
-- and can be different from the cabal-install (the tool) version and the
-- Cabal (the library) version you are using. As such, the Cabal (the library)
-- version used must be equal or greater than the version stated in this field.
-- Starting from the specification version 2.2, the cabal-version field must be
-- the first thing in the cabal file.

-- Initial package description 'tmp' generated by
-- 'cabal init'. For further documentation, see:
--   http://haskell.org/cabal/users-guide/
--
-- The name of the package.
name:               tmp

-- The package version.
-- See the Haskell package versioning policy (PVP) for standards
-- guiding when and how versions should be incremented.
-- https://pvp.haskell.org
-- PVP summary:     +-+------- breaking API changes
--                  | | +----- non-breaking API additions
--                  | | | +--- code changes with no API change
version:            0.1.0.0

-- A short (one-line) description of the package.
-- synopsis:

-- A longer description of the package.
-- description:

-- The license under which the package is released.
license:            BSD-3-Clause

-- The file containing the license text.
license-file:       LICENSE

-- The package author(s).
author:             hogehoge

-- An email address to which users can send suggestions, bug reports, and patches.
maintainer:         fumufumu@example.com

-- A copyright notice.
-- copyright:
build-type:         Simple

-- Extra doc files to be distributed with the package, such as a CHANGELOG or a README.
extra-doc-files:    CHANGELOG.md

-- Extra source files to be distributed with the package, such as examples, or a tutorial module.
-- extra-source-files:

common warnings
    ghc-options: -Wall

executable tmp
    -- Import common warning flags.
    import:           warnings

    -- .hs or .lhs file containing the Main module.
    main-is:          Main.hs

    -- Modules included in this executable, other than Main.
    -- other-modules:

    -- LANGUAGE extensions used by modules in this package.
    -- other-extensions:

    -- Other library packages from which modules are imported.
    build-depends:    base ^>= 4.20.0.0,
                      template-haskell,
		      matrix,
		      string-qq,
		      ghcjs-dom,
		      reflex-dom

    -- Directories containing source files.
    hs-source-dirs:   app

    -- Base language which the package is written in.
    default-language: Haskell2010
```

Reflex-dom/GHCJS-dom 9.10 有志によるCompatibility Patch (ghcjs-domは0.9.9.2ですでにマージされ修正済みっぽいが、、reflexのほうはまだみたい。): <https://github.com/amesgen/reflex-dom> <https://github.com/ymeister/ghcjs-dom.git>

（追記）Ormolu Live: <https://github.com/tweag/ormolu/tree/master/ormolu-live>  
こちらもnixを使ってはいるものの、ビルド自体は`cabal build`で行われていて、パッケージマネージャとしては使えていない模様、、、
