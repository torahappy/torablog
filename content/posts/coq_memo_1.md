---
title: "Coqメモ"
date: 2019-05-01T10:21:00+09:00
tags: ["Coq"]
---

Coqのすごいのは、inductionといって何かしらの概念の定義（どういったものを用意するとそれが作れるか）から、自動的にそれにまつわる公理が定義されるところ  
これは自然数の帰納法を一般化したもので、例えば自然数の定義はCoqでは

```coq
Inductive nat : Set :=
  | O : nat
  | S : nat -> nat.
```

（意味: Oというnatを生成する関数と、なんらかのnatからnatを生成する関数がある）

だけど、そこから自動的に

```coq
nat_rect : forall P : nat -> Type,
       P 0 ->
       (forall n : nat, P n -> P (S n)) -> forall n : nat, P n
net_rec : forall P : nat -> Set,
       P 0 ->
       (forall n : nat, P n -> P (S n)) -> forall n : nat, P n
```

が定義される。  
集合の総和を表すFamilyUnionでは、定義

```coq
Inductive FamilyUnion: Ensemble T :=
  | family_union_intro: forall (S:Ensemble T) (x:T),
    In F S -> In S x -> In FamilyUnion x.
```

に対して

```coq
FamilyUnion_ind : forall (T : Type) (F : Family T) (P : T -> Prop),
       (forall (S : Ensemble T) (x : T),
        In F S -> In S x -> P x) ->
       forall t : T, FamilyUnion F t -> P t
```

が定義される。  
Coqをはじめとする関数型言語ではFamilyUnionやnatのような複数のものから作り出されたものを"分解"できる。だからfamily_union_introからそれを作り出したIn F S や In S x を取り出すことができる。  
この仕組みを使ってFamilyUnion_indが定義されている。

自然数は帰納的に（自分自身で自分自身を定義して）定義されているから、  
ある数nから、「kで成り立つならば、k+1で成り立つ」を使って、n-1で成り立っているならばnで成り立つ、n-2で成り立っているならばn-1で成り立つ、...というのを0になるまで繰り返して、最終的に「0で成り立つ」を使って証明する。この方法だと無限の長さの証明をしなきゃいけない気がして正しいかは微妙だけど、どのような数が与えられても遡って0に至るまでは有限の時間で計算可能だから正しいということになる。つまり、（当たり前の話だけど）有限の時間で計算可能＝有限の時間で証明が構築可能ならば、その証明が正しいというのがCoqにおける帰納法（？）の公理ということになる。

ただし、自然数一つひとつをしらみつぶしに証明していくと結局無限の時間がかかってしまうから、厳密に言えば、「Forall x. (P(x)に対する有限の証明) -> (Forall x. (P(x))の証明」が公理（つまり交換可能性！）とも言える。

この公理のすごいのは集合の濃度に関しては聞かれていないことだ。証明を構成するコンストラクタ（`O`や`S`など）の世界の話だから。
