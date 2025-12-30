---
title: "色々メモその１"
date: 2018-01-13T19:54:01+09:00
description: "Haskell、圏論、自然変換、モノイド、とかに関するメモ。「Haskellのforallは三通りの解釈ができる。」「任意のモノイドは適当な集合AについてEnd(A)の部分モノイドと同型であることの証明」"
---

<http://nineties.github.io/category-seminar/>  
これを読んだらちょっとだけ圏論がわかった気がする。

### Haskellのforallは三通りの解釈ができる。

(1)プログラム的解釈・・・ポリモルフィズム
   
(2)圏論的解釈・・・自然変換
   
(3)論理学的解釈・・・全称量化子

てことでいいのかな？



### 任意のモノイドは適当な集合AについてEnd(A)の部分モノイドと同型であることの証明

モノイド$(M, *)$について、
$A = M$ と定める。

また、
$B$は$End(A)$の部分モノイドとし、その要素は
$B = \\left\\{ f | m ∈ M, f(x) = m*x\\right\\}$ と定める。
($B ⊆ End(M) = End(A)$)

モノイド$(M, *)$と、モノイド$B$が同型であることを示す。

(1)

ここでは可読性のため、$F(m) = Fm, F(n) = Fn$と表記する。

$F : B \rightarrow M, Fm(x) = m*x$ とする。

<p>
$
\begin{aligned}
(F(m*n))(x) &= m*n*x \\
            &= m*(n*x) \\
            &= m*Fn(x) \\
            &= Fm(Fn(x)) \\
            &= (Fm \circ Fn)(x) \\
\end{aligned}
$
<br>
$∴ F(m*n) = F(m) \circ F(n)$
</p>

したがって、$(M, *)$から$B$への準同型が存在する。

(2)

$e$が$M$の単位元で、$G : B \rightarrow M, G(f) = f(e)$とすると、

<p>
$
\begin{aligned}
G(f \circ g)& = (f \circ g)(e) \\
& = f(g(e)) \\
& = f(m*e) & (m ∈ M) \\
& = f(m) \\
& = n*m & (n ∈ M) \\
& = n*e*m*e \\
& = f(e)*g(e) \\
& = G(f)*G(g) \\
\end{aligned}
$
<br>
$\therefore G(f \circ g) = G(f)*G(g)$
</p>

したがって、$B$から$(M, *)$への準同型が存在する。


以上より、モノイド$(M, *)$とモノイド$B$は同型。
したがって、任意のモノイドは適当な集合Aについて$End(A)$の部分モノイドと同型である。∎