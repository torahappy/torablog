---
title: "Haskell Proofs"
date: 2017-12-20T01:52:01+09:00
description: "実はHaskellで定理証明のようなことができる。"
tags: ["haskell", "theorem proving"]
---

{{< highlight haskell >}}
{-# LANGUAGE RankNTypes #-}

-- [Definitions]
-- Not
newtype Not p = Not (forall q. p -> q)

-- [Axioms]
-- Law of excluded middle
classic :: Either p (Not p)
classic = undefined

-- [Proofs]
-- Syllogism
syllogism :: (p -> q) -> (q -> r) -> (p -> r)
syllogism h1 h2 = h2 . h1

-- Commutative law (And)
commutativeLawAnd :: (p, q) -> (q, p)
commutativeLawAnd (p, q) = (q, p)

-- Commutative law (Or)
commutativeLawOr :: Either p q -> Either q p
commutativeLawOr = either Right Left

-- Associative law
associativeLaw :: ((p -> q) -> r) -> (p -> (q -> r))
associativeLaw h = const $ h . const

-- P -> Not P -> Q
bomb :: p -> Not p -> q
bomb p (Not pq) = pq p

-- (P and (Not P)) -> Q
bombAnd :: (p, Not p) -> q
bombAnd (p, np) = bomb p np

-- De Morgan
deMorgan :: (Not p, Not q) -> Not (Either p q)
deMorgan (np, nq) = Not $ either (`bomb` np) (`bomb` nq)

-- Double negative
pnnp :: p -> Not (Not p)
pnnp p = Not $ bomb p

nnpp :: Not (Not p) -> p
nnpp nnp = either id (`bomb` nnp) classic

-- contraposition
contraposition1 :: (p -> q) -> (Not q -> Not p)
contraposition1 h nq = Not (\p -> bomb (h p) nq)

contraposition2 :: (Not q -> Not p) -> (p -> q)
contraposition2 h p = nnpp $ contraposition1 h (pnnp p)

-- implyToOr
implyToOr :: (p -> q) -> Either (Not p) q
implyToOr pq = either (Right . pq) Left classic

-- Peirce's Law
peirce :: ((p -> q) -> p) -> p
peirce h = either id (\(Not px) -> h px) classic

-- some props
prop1 :: (forall p. ((p -> q) -> q)) -> ((p -> q) -> p) -> p
prop1 h h0 = h0 $ const $ h id

{{< /highlight >}}

Haskellで定理証明もどき、やってみた

### どうしてHaskellで定理証明ができるの？
Haskellみたいな言語だと、型のワイルドカード的なものがある。それは、数学のようにaやbなどの文字で表す。

例えば、`a -> b`という型のaやbにはInt型でもString型でも、何でも入る。
この時、この関数に型aの引数を入れると型bの値が返ってくるが、これは、
```
aが正しいならばbが正しい
aが正しい
従って、bが正しい
```
という推論（モーダスポネンス）と同じことだと考えることができる。

ほかにも、aかbどちらかの値を表す`Either a b`は「aまたはb」、aとb両方の値を保持する`(a, b)`は「aかつb」などと、論理の世界とプログラムの世界には対応がある。

### 解説

{{< highlight haskell >}}
-- [定義]
-- Not
newtype Not p = Not (forall q. p -> q)
{{< /highlight >}}
Haskellの世界には否定は存在しないので、ここで定義している。

forallは、型のワイルドカード的なやつである。例えば、型`forall a. a -> Int`の関数には文字列でも数値でも関数でも、とにかくどんな型を入れても良い。その代わり、この関数は定数関数以外ありえない。なぜなら、もし引数と返り値の間に関係性があったならば、その関係性によって引数の型を制限してしまうからである。我々は引数についてのいかなる情報も得ることはできない。だから、このような型を持てるのは`const 3`などの定数関数のみである。

`forall q. p -> q`というのは「pからなんでも出てくるような関数」の型なので、pが存在してしまうと、そのような関数は存在できなくなる。つまり、`forall q. p -> q`という関数が存在するということは、pでないということだ、と直感的には理解できる。

実際、この定義で後々の否定に関する定理も証明できるし、論理的にも、$(\forall q. p \Rightarrow q) \Leftrightarrow (\lnot p)$である。

Haskellには否定の概念はないが、多相型(forall)があるので、否定を表現することができる。Haskellはこんな調子で、何か有名な概念そのものは言語に実装されていないが、代わりにその概念を定義できるような機能が実装されている、というパターンが多い。モナドや関手だって型クラスの機能を使ってHaskellで定義されているし、リストのデータ構造の定義もHaskellで書かれている。

{{< highlight haskell >}}
-- [公理]
-- 排中律
classic :: Either p (Not p)
classic = undefined
{{< /highlight >}}

undefinedとは、`forall a. a`の型を持つ関数である。この関数は、評価された瞬間にエラーを出力してプログラムを終了させる。（だから、型が`forall a. a`でも問題ないと考えることもできるかも。）

undefinedは何にでもなれる。Int型の関数の定義をundefinedにしてもいいし、String型の関数の定義をundefinedにしてもいい。何にもなれる関数が存在しているというのは、論理的に考えれば「全ての命題が正しい」ということである。

また、undefinedを使わなかったとしても、関数が再帰的な定義をされていると、型と命題の対応が変になる。例えば、`y x = x (y x)`で定義される関数の型は、`forall t. (t -> t) -> t`である。「「tならばt」ならばt」これは論理的には正しくない。でも、循環定義をすると作れちゃうのである。さらに驚くべきことに、`x = x`で定義される関数xの型は`forall a. a`である。

排中律の定義がundefinedになっていること、undefinedの型の論理的解釈は「全ての命題が正しい」、再帰的な定義をすると型と命題の対応が変になること、これらは全て、「Haskellにはundefinedや停止しない関数などの、値が決まらない関数が存在できる」という話とつながっているのかもしれない。本来、排中律は「pまたはpでない」という法則のことだが、プログラム的に考えると、エラーなどで、pであるかpでないかが決まらないこともあるはずだ。つまり、そもそもHaskellにおいては排中律は成り立たないのかもしれない。うーん...

{{< highlight haskell >}}
-- [証明]
-- 三段論法
syllogism :: (p -> q) -> (q -> r) -> (p -> r)
syllogism h1 h2 = h2 . h1

-- 交換法則（かつ）
commutativeLawAnd :: (p, q) -> (q, p)
commutativeLawAnd (p, q) = (q, p)

-- 交換法則（または）
commutativeLawOr :: Either p q -> Either q p
commutativeLawOr = either Right Left

-- 結合法則
associativeLaw :: ((p -> q) -> r) -> (p -> (q -> r))
associativeLaw h = const $ h . const
{{< /highlight >}}
Notを使わない定理たち。

{{< highlight haskell >}}
-- P -> Pでない -> Q
bomb :: p -> Not p -> q
bomb p (Not pq) = pq p

-- (Pかつ(Pでない))ならばQ
bombAnd :: (p, Not p) -> q
bombAnd (p, np) = bomb p np

-- ド・モルガンの法則
deMorgan :: (Not p, Not q) -> Not (Either p q)
deMorgan (np, nq) = Not $ either (`bomb` np) (`bomb` nq)

-- 二重否定法則
pnnp :: p -> Not (Not p)
pnnp p = Not $ bomb p

nnpp :: Not (Not p) -> p
nnpp nnp = either id (`bomb` nnp) classic

-- 対偶
contraposition1 :: (p -> q) -> (Not q -> Not p)
contraposition1 h nq = Not (\p -> bomb (h p) nq)

contraposition2 :: (Not q -> Not p) -> (p -> q)
contraposition2 h p = nnpp $ contraposition1 h (pnnp p)

--いろいろ
implyToOr :: (p -> q) -> Either (Not p) q
implyToOr pq = either (Right . pq) Left classic
{{< /highlight >}}
Notを使う定理たち。ここにあるのはどれも重要な否定に関する定理だが、あのNotの定義でちゃんと証明できるのである。すごい！

nnppとimplyToOrは公理classicを使ってしまっているが、これらは論理的にも排中律がないと成り立たない規則である。ソースは[Coq](https://coq.inria.fr/)のソースコード。

{{< highlight haskell >}}
-- パースの法則
peirce :: ((p -> q) -> p) -> p
peirce h = either id (\(Not px) -> h px) classic
{{< /highlight >}}
パースの法則。排中律がないと証明できないのにNotを使っていない、という謎な法則である。たしか排中律と同値なのでこれをごにょごにょ変形していくと`Either p (forall q. p -> q)`となるのだろう。誰かやって。