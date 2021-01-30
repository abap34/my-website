---
title: "Juliaで A .+ Bをした時、何が起きているのか"
emoji: "🤮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["julia"]
published: false
---

# 自己紹介

{ここに自己紹介を書く}

普段は機械学習諸々を勉強していて、https://github.com/abap34/DeepShiba.jl を作ったりしています。(最近はあまり手が回っていないですが...)

この過程でブロードキャスト周りを制御する必要があり、 Juliaの ~~ソースコードを読むハメになった~~ ソースコードを読ませていただく機会をいただけたので、記事にして残そうと思います。


# この記事について

タイトルの通り、Juliaの実装を追うことによって、Juliaで`A .+ B`を実行した時に何が起きているのかを追う記事です。

また、その過程で必要な諸概念についても解説していきます。

ただし、あくまでJuliaのブロードキャストの実装を追う記事なので、そのものについては解説しません。
(つまり、Juliaとブロードキャストの基礎の基礎がわかっていることが前提となっています。)


**また、記事の作成にあたってはなるべく正確を期するよう努力していますが、もし間違いがある場合はコメントをしていただけると嬉しいです。**
特にコアな部分の話や翻訳についてはあまり自信がないので、優しいマサカリをお待ちしています。

## 前提

環境は以下の通りです。

```
Julia Version 1.5.2
Commit 539f3ce943 (2020-09-23 23:17 UTC)
Platform Info:
  OS: macOS (x86_64-apple-darwin18.7.0)
  CPU: Intel(R) Core(TM) i5-8259U CPU @ 2.30GHz
  WORD_SIZE: 64
  LIBM: libopenlibm
  LLVM: libLLVM-9.0.1 (ORCJIT, skylake)
```

:::message
Juliaでは、`versioninfo()`で現在の環境を表示できます。
:::

また、 `A .+ B`について、具体的にはこれらを指します。(あくまで具体的な値を扱った方がわかりやすい、という趣旨なのでこれ自体に意味はありません)

```julia
julia> A = [1 2 3
           4 5 6
           7 8 9]
3×3 Array{Int64,2}:
 1  2  3
 4  5  6
 7  8  9

julia> B = [1 2 3]
1×3 Array{Int64,2}:
 1  2  3

julia> A .+ B
3×3 Array{Int64,2}:
 2   4   6
 5   7   9
 8  10  12

```




# ブロードキャスト

さて、タイトルになっている 
`A .+ B` について　少し解説します。

これは、「ブロードキャスト」と呼ばれる機能を使用しています。

ブロードキャストとは、多次元配列どうしの演算などに用いられる機能で、これを利用して簡潔な記述が可能です。(詳しくは、[公式ドキュメント](https://docs.julialang.org/en/v1/manual/arrays/#Broadcasting)を参照してください)

他の言語でも多く実装されていて、特にNumpyが有名です。

Juliaでは関数を呼び出すときに、その関数名の末尾に `.` をつけることでブロードキャストを利用できます。

```julia
# broadcastの例

julia> f(x) = 2x
f (generic function with 1 method)

julia> arr = [2, 4, 5]
3-element Array{Int64,1}:
 2
 4
 5

julia> f.(arr)
3-element Array{Int64,1}:
  4
  8
 10
```

また、演算子に対しては`A .op B`のような記法でブロードキャストが利用可能です。(記事のタイトルを参照してください！)

:::details [補足]:演算子について
Juliaでは、様々な算術演算子を扱えます。
ただしこれらは単に特殊な記法が可能というだけで、通常のJuliaの関数と変わりありません。
なので
```julia
julia> +(4, 5)
9
```
のように通常のメソッド呼び出しの記法が使えますし、
**関数は第一級オブジェクトなので、引数としても使えます。**
演算子のような記法(infix notation)が使える関数名については、[Scheme製のJuliaのパーサのソースコード](https://github.com/JuliaLang/julia/blob/master/src/julia-parser.scm)を参考にしてください。
(とは言え、突然謎の演算子が登場するコードの可読性は最悪ですが....)
:::

:::message
この記事では、`+`, `-`, `*`, `/` などの`A + B`のような記法(infix notation)が可能な関数を演算子と呼びます。
:::

# コードを追う

## 第一段階

ところで、Juliaのコードはコンパイルされる時、いくつかの段階を経て機械語へと翻訳されます。
具体的には、
ソースコード -> 低レベル表現 -> 型付表現 -> LLVM中間表現 -> ネイティブコード
という順です。
そして、その段階での表現は、それぞれ

- @code_lowerd(低レベル表現),
- @code_typed(型付表現)
- @code_llvm(LLVM中間表現)
- @code_native(ネイティブコード)
 
で確認することができます。

では早速使ってみましょう。

```
julia> f() = A .+ B
f (generic function with 2 methods)

julia> @code_lowered f()
CodeInfo(
1 ─ %1 = Base.broadcasted(Main.:+, Main.A, Main.B)
│   %2 = Base.materialize(%1)
└──      return %2
)
```

:::message alert
ここでわざわざ関数を定義していますが、これはそのまま A .+ Bとすると何故かエラーが出るためです。***Julia Master*** の方はどうしてそうなるのか教えてくださると嬉しいです。
:::

さて、f()はブロードキャストした演算結果を返すだけですが、どうやら2回何かが計算されているようです。

早速それらを詳しくみても良いのですが、ひとまず簡単に2つの関数を確認してみます。

Base.broadcastedはどうみてもbroadcastをしていますが、Base.materializeは何をしているんでしょう。確認してみます。

:::message
Juliaの対話的環境では、
`?hoge`でドキュメントを参照することができます。
:::

```

help?> Base.materialize
  Broadcast.materialize(bc)

  Take a lazy Broadcasted object and compute the result

```

>Take a lazy Broadcasted object and compute the result
 
なるほど、どうやらbroadcastは実際にその場で配列を複製したりするのではなく、遅延評価を行うことで高速に動作するようですね！
これはソースコードを読むことでいろいろな知見が得られそうです。

それでは`Base.broadcasted`を確認しましょう！


## Base.broadcasted

では早速`?Base.broadcasted`を実行してみてください^^

:::details 取材班を襲った悲劇
```
help?> Base.broadcasted
  No documentation found.

  Base.Broadcast.broadcasted is a Function.

  # 53 methods for generic function "broadcasted":
  [1] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(+), r::AbstractUnitRange, x::Real) in Base.Broadcast at broadcast.jl:1038
  [2] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(+), r::OrdinalRange) in Base.Broadcast at broadcast.jl:1029
  [3] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(+), r::StepRangeLen) in Base.Broadcast at broadcast.jl:1030
  [4] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(+), r::LinRange) in Base.Broadcast at broadcast.jl:1031
  [5] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(+), x::Real, r::AbstractUnitRange) in Base.Broadcast at broadcast.jl:1037
  [6] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(+), r::StepRangeLen{T,R,S} where S where R, x::Number) where T in Base.Broadcast at broadcast.jl:1042
  [7] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(+), r::LinRange, x::Number) in Base.Broadcast at broadcast.jl:1046
  [8] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(+), r::AbstractRange, x::Number) in Base.Broadcast at broadcast.jl:1040
  [9] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(+), x::Number, r::StepRangeLen{T,R,S} where S where R) where T in Base.Broadcast at broadcast.jl:1044
  [10] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(+), x::Number, r::LinRange) in Base.Broadcast at broadcast.jl:1047
  [11] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(+), x::Number, r::AbstractRange) in Base.Broadcast at broadcast.jl:1041
  [12] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(+), r1::AbstractRange, r2::AbstractRange) in Base.Broadcast at broadcast.jl:1048
  [13] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(-), r::AbstractUnitRange, x::Number) in Base.Broadcast at broadcast.jl:1050
  [14] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(-), r::OrdinalRange) in Base.Broadcast at broadcast.jl:1033
  [15] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(-), r::StepRangeLen) in Base.Broadcast at broadcast.jl:1034
  [16] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(-), r::LinRange) in Base.Broadcast at broadcast.jl:1035
  [17] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(-), r::StepRangeLen{T,R,S} where S where R, x::Number) where T in Base.Broadcast at broadcast.jl:1053
  [18] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(-), r::LinRange, x::Number) in Base.Broadcast at broadcast.jl:1057
  [19] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(-), r::AbstractRange, x::Number) in Base.Broadcast at broadcast.jl:1051
  [20] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(-), x::Number, r::StepRangeLen{T,R,S} where S where R) where T in Base.Broadcast at broadcast.jl:1055
  [21] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(-), x::Number, r::LinRange) in Base.Broadcast at broadcast.jl:1058
  [22] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(-), x::Number, r::AbstractRange) in Base.Broadcast at broadcast.jl:1052
  [23] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(-), r1::AbstractRange, r2::AbstractRange) in Base.Broadcast at broadcast.jl:1059
  [24] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(*), x::Number, r::StepRangeLen{T,R,S} where S where R) where T in Base.Broadcast at broadcast.jl:1062
  [25] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(*), x::Number, r::LinRange) in Base.Broadcast at broadcast.jl:1064
  [26] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(*), x::Number, r::AbstractRange) in Base.Broadcast at broadcast.jl:1061
  [27] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(*), r::StepRangeLen{T,R,S} where S where R, x::Number) where T in Base.Broadcast at broadcast.jl:1067
  [28] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(*), r::LinRange, x::Number) in Base.Broadcast at broadcast.jl:1069
  [29] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(*), r::AbstractRange, x::Number) in Base.Broadcast at broadcast.jl:1066
  [30] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(/), r::StepRangeLen{T,R,S} where S where R, x::Number) where T in Base.Broadcast at broadcast.jl:1072
  [31] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(/), r::LinRange, x::Number) in Base.Broadcast at broadcast.jl:1074
  [32] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(/), r::AbstractRange, x::Number) in Base.Broadcast at broadcast.jl:1071
  [33] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(\), x::Number, r::StepRangeLen) in Base.Broadcast at broadcast.jl:1077
  [34] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(\), x::Number, r::LinRange) in Base.Broadcast at broadcast.jl:1078
  [35] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(\), x::Number, r::AbstractRange) in Base.Broadcast at broadcast.jl:1076
  [36] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(big), r::UnitRange) in Base.Broadcast at broadcast.jl:1080
  [37] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(big), r::StepRange) in Base.Broadcast at broadcast.jl:1081
  [38] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(big), r::StepRangeLen) in Base.Broadcast at broadcast.jl:1082
  [39] broadcasted(::Base.Broadcast.DefaultArrayStyle{1}, ::typeof(big), r::LinRange) in Base.Broadcast at broadcast.jl:1083
  [40] broadcasted(::typeof(+), I::CartesianIndices{N,R} where R<:Tuple{Vararg{AbstractUnitRange{Int64},N}}, j::CartesianIndex{N}) where N in Base.Broadcast at broadcast.jl:1086
  [41] broadcasted(::typeof(+), j::CartesianIndex{N}, I::CartesianIndices{N,R} where R<:Tuple{Vararg{AbstractUnitRange{Int64},N}}) where N in Base.Broadcast at broadcast.jl:1088
  [42] broadcasted(::typeof(-), I::CartesianIndices{N,R} where R<:Tuple{Vararg{AbstractUnitRange{Int64},N}}, j::CartesianIndex{N}) where N in Base.Broadcast at broadcast.jl:1090
  [43] broadcasted(::typeof(-), j::CartesianIndex{N}, I::CartesianIndices{N,R} where R<:Tuple{Vararg{AbstractUnitRange{Int64},N}}) where N in Base.Broadcast at broadcast.jl:1093
  [44] broadcasted(::typeof(LinearAlgebra.:*ₛ), out, beta) in LinearAlgebra at /Applications/Julia-1.4.app/Contents/Resources/julia/share/julia/stdlib/v1.4/LinearAlgebra/src/diagonal.jl:290
  [45] broadcasted(::typeof(*), x::Number, J::LinearAlgebra.UniformScaling) in LinearAlgebra at /Applications/Julia-1.4.app/Contents/Resources/julia/share/julia/stdlib/v1.4/LinearAlgebra/src/uniformscaling.jl:227
  [46] broadcasted(::typeof(*), J::LinearAlgebra.UniformScaling, x::Number) in LinearAlgebra at /Applications/Julia-1.4.app/Contents/Resources/julia/share/julia/stdlib/v1.4/LinearAlgebra/src/uniformscaling.jl:228
  [47] broadcasted(::typeof(/), J::LinearAlgebra.UniformScaling, x::Number) in LinearAlgebra at /Applications/Julia-1.4.app/Contents/Resources/julia/share/julia/stdlib/v1.4/LinearAlgebra/src/uniformscaling.jl:230
  [48] broadcasted(::typeof(^), J::LinearAlgebra.UniformScaling, x::Number) in LinearAlgebra at /Applications/Julia-1.4.app/Contents/Resources/julia/share/julia/stdlib/v1.4/LinearAlgebra/src/uniformscaling.jl:235
  [49] broadcasted(::typeof(Base.literal_pow), ::typeof(^), J::LinearAlgebra.UniformScaling, x::Val) in LinearAlgebra at /Applications/Julia-1.4.app/Contents/Resources/julia/share/julia/stdlib/v1.4/LinearAlgebra/src/uniformscaling.jl:237
  [50] broadcasted(::S, f, args...) where S<:Base.Broadcast.BroadcastStyle in Base.Broadcast at broadcast.jl:1240
  [51] broadcasted(f, arg1, arg2, args...) in Base.Broadcast at broadcast.jl:1235
  [52] broadcasted(f, arg1, args...) in Base.Broadcast at broadcast.jl:1230
  [53] broadcasted(f, args...) in Base.Broadcast at broadcast.jl:1222
```
:::


これのせいでmarkdownファイルがスクロールしにくくなりました。Julia言語を許さない。

さて、このようにドキュメントでもその関数がよくわからない時はソースコードを読みにいきましょう！！！！！！！！！

そんな時のためにJuliaは便利なマクロを標準で搭載しています。

```
julia> @functionloc Base.broadcasted(Main.:+, Main.A, Main.B)
("/path/to/julia/bin/../share/julia/base/broadcast.jl", 1235)
```

@functionlocを使用することで、呼び出された関数がどこで定義されているのか確認することができます。

というわけで、base/broadcast.jlを見てみましょう。

Juliaは大学とかでやたら使われている某言語などと違ってオープンソースなので、ソースコードはGitHubで確認できます。(が、Juliaはどんどん改善されていくので、素直に手元にあるコードを確認した方が良いです。該当するブランチを探すのも面倒ですし, エディタの機能を色々使えて便利です。
broadcast.jlは、 `/path/to/Julia-1.5/Contents/Resources/julia/share/julia/base にあります。

さて、1259行目を参照してください。

```julia
@inline function broadcasted(f, arg1, arg2, args...)
    arg1′ = broadcastable(arg1)
    arg2′ = broadcastable(arg2)
    args′ = map(broadcastable, args)
    broadcasted(combine_styles(arg1′, arg2′, args′...), f, arg1′, arg2′, args′...)
end
```


これを素直にグラフにするとこんな感じです。

![](https://storage.googleapis.com/zenn-user-upload/i8bgv2ystdt27lqmvtxuq2l2lnpz)

これにどんどん情報を付け足して完成を目指します。

さて、broadcastedの中でbroadcastedが呼ばれていることにお気づきでしょうか？
他の言語を使っている方だと(この記事を読んでいる方にいるかはわかりませんが...)

「え？再帰なのにベースケースなくない？これ無限ループでは.....？」

となっているかもしれません。
しかし最後の行で呼ばれる`broadcasted`は、同じ関数ですが別のメソッドです。Juliaでは、「多重ディスパッチ」という仕組みによって、動的にメソッドが呼び出されます。

この多重ディスパッチを用いることで、数学的に自然な考え方で柔軟な設計が可能です。(しかもこの動的なディスパッチはほとんどゼロコストです！Juliaコンパイラ万歳！)


:::details 多重ディスパッチの簡単な例
```julia
julia> double(x::Int) = 2x
double (generic function with 1 method)

julia> double(x::String) = x * x  # Juliaの文字列結合は * です
double (generic function with 2 methods)

julia> double(10)
20

julia> double("JuliaLang")
"JuliaLangJuliaLang"
```
:::

さて、では実際に関数の中身を見ていこうと思います。

### broadcastable
まずは1~3行目です。
引数を順番に`broadcastable`に渡しているようです。
3引数目までは順番に、それ以降は`map`に渡しているのがよくわからないですね...
ほとんどの場合、引数は2個ですから、(むしろ3個以上の場合が思いつかない) `map`のオーバーヘッドを気にしているんでしょうか。

まずは`broadcastable`を確認してみましょう。

Broadcast.broadcastable
```julia
  Broadcast.broadcastable(x)

  Return either x or an object like x such that it supports axes, indexing, and its type supports ndims.
  # 軸、インデックス、次元数をサポートするようなx(もしくはxのようなオブジェクト)を返します。
  If x supports iteration, the returned value should have the same axes and indexing behaviors as collect(x).
  # もしxがiterationをサポートしている場合は、返り値はcollect(x)の
  If x is not an AbstractArray but it supports axes, indexing, and its type supports ndims, then broadcastable(::typeof(x)) may be
  implemented to just return itself. Further, if x defines its own BroadcastStyle, then it must define its broadcastable method to
  return itself for the custom style to have any effect.

  Examples
  ≡≡≡≡≡≡≡≡≡≡

  julia> Broadcast.broadcastable([1,2,3]) # like `identity` since arrays already support axes and indexing
  3-element Array{Int64,1}:
   1
   2
   3
  
  julia> Broadcast.broadcastable(Int) # Types don't support axes, indexing, or iteration but are commonly used as scalars
  Base.RefValue{Type{Int64}}(Int64)
  
  julia> Broadcast.broadcastable("hello") # Strings break convention of matching iteration and act like a scalar instead
  Base.RefValue{String}("hello")
```

:::

`broadcastable`は引数を次元や軸を持つ形に変形します。今回はAもBも配列なので、変化は起きないですね。



```julia
julia> Broadcast.broadcastable(A) == A
true

julia> Broadcast.broadcastable(B) == B
true
```

これをグラフに追記しておきます。(返り値の型をグラフのエッジに書くことにします)

![](https://storage.googleapis.com/zenn-user-upload/u03iel4vdn3rrvhf0yofes65w9o8)


### combine_styles


次に`combine_styles`を確認してみます。

:::details combine_styles
```julia
help?> Broadcast.combine_styles
  combine_styles(cs...) -> BroadcastStyle

  Decides which BroadcastStyle to use for any number of value arguments. Uses BroadcastStyle to get the style for each argument,
  and uses result_style to combine styles.

  Examples
  ≡≡≡≡≡≡≡≡≡≡

  julia> Broadcast.combine_styles([1], [1 2; 3 4])
  Base.Broadcast.DefaultArrayStyle{2}()
```
:::


`BroadcastStyle`を返す関数です。
そもそも`BroadcastStyle`とは何か、確認してみます。

BroadcastStyle

```julia
help?> Broadcast.BroadcastStyle
  BroadcastStyle is an abstract type and trait-function used to determine behavior of objects under broadcasting.
# `BroadcastStyle`はabstract typeであり、ブロードキャストされるオブジェクトの振る舞いを決定するためのtrait-functionです。
  BroadcastStyle(typeof(x)) returns the style associated with x. To customize the broadcasting behavior of a type, one can declare
  a style by defining a type/method pair
# BroadcastStyle(typeof(x))はxに関連付けられたスタイルを返します。型のブロードキャスト動作をカスタマイズするために、型とメソッドのペアを定義することでスタイルを宣言できます。
  struct MyContainerStyle <: BroadcastStyle end
  Base.BroadcastStyle(::Type{<:MyContainer}) = MyContainerStyle()

  One then writes method(s) (at least similar) operating on Broadcasted{MyContainerStyle}. There are also several pre-defined
  subtypes of BroadcastStyle that you may be able to leverage; see the Interfaces chapter for more information.

  ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

  BroadcastStyle(::Style1, ::Style2) = Style3()

  Indicate how to resolve different BroadcastStyles. For example,

  BroadcastStyle(::Primary, ::Secondary) = Primary()

  would indicate that style Primary has precedence over Secondary. You do not have to (and generally should not) define both
  argument orders. The result does not have to be one of the input arguments, it could be a third type.

  Please see the Interfaces chapter of the manual for more information.
```







:::details [補足]:abstract typeについて
Juliaは動的型付けの言語ですが、静的型付けのシステムの利点をある程度取り入れて効率的なコードを生成しています。
Juliaの型には大別してabstract type, concrete typeの二種類があります。(前者は抽象型、後者は具体型、もしくは具象型と訳されます)
オブジェクトが実際に持つ型が具体型で、(例:Int64, String)
全ての具体型は抽象型を親(supertype)に持ちます。(例:Int64の親タイプはSigned(符号付き整数))
つまりJuliaは次のようにそれぞれの型が階層関係を持ち、全ての型はAnyの子です。
![](https://storage.googleapis.com/zenn-user-upload/ne8vtmy4j80n7uakbldqasm2wo75)[^1]
[^1]:https://www.sas.upenn.edu/~jesusfv/Chapter_HPC_8_Julia.pdf

ちなみにある抽象型の子は`subtypes(Hoge)`,また具体型の親は`supertype(Huga)`で確認できます。
また、AがBの子に当たるかどうかは、 `A <: B`で判定することができます。

:::


# trait-functionについて
trait-functionとは、Holy traitと呼ばれる設計手法で用いられる関数です。
Holy-traitについて簡単に説明してみます。[^2]
[^2]:こちらの記事を元にして説明しています。(https://qiita.com/tenfu2tea/items/09c3528b2144d1eac74a)

次のような関数を実装することを考えてみます。(あくまで例でなので、より効率的な実装などがありますが、コードの内容についてはあまり突っ込まないでくださると嬉しいです。)

- 引数がタプルやベクトル（１次元配列）なら、それ自身を返す。
- 引数が数（スカラー）なら、1つの要素のベクトルを作って返す。

(つまりは、引数を繰り返し可能にする関数です！)


さて、これを**とても素直に**実装すると次のようになります。

```julia
# var isa Hoge は、 typeof(var) <: Hogeと等価です。
julia> function aslist(var)
           if var isa Number 
               return [var,]
           elseif var isa Tuple || var isa Vector
               return var
           end
       end
aslist (generic function with 1 method)

julia> aslist(10)
1-element Array{Int64,1}:
 10

julia> aslist([1, 2, 3])
3-element Array{Int64,1}:
 1
 2
 3

julia> aslist((1, 2, 3))
(1, 2, 3)
```

ですがこれはとても悪いコードです。
Juliaでは多重ディスパッチを用いて以下のように書かれるべきです。

```julia
julia> aslist(x::Int) = [x, ] 
aslist (generic function with 2 methods)

julia> aslist(arr::Vector) = arr
aslist (generic function with 3 methods)

julia> aslist(t::Tuple) = t
aslist (generic function with 4 methods)
```

ところで、このとき引数が`Vector`, `Tuple`のときは同じ処理をしているので、これをまとめたくなりますね。
そのときは、`Union` 型を用いると便利です。
`Union`型は、特殊なabstract typeの一種で、複数の型の和集合を表します。

例えば、`Int`と`String`をまとめた型は、

```julia
julia> IntOrString = Union{Int, String}
Union{Int64, String}
```

と宣言できます。

したがって

```julia
julia> Int <: IntOrString
true

julia> String <: IntOrString
true
```

なので、次のようなコードを書くことができます。


```julia
julia> f(x::IntOrString) = println("nice")
f (generic function with 2 methods)

julia> f(1)
nice

julia> f("Hi")
nice

julia> f([1, 2, 3])
ERROR: MethodError: no method matching f(::Array{Int64,1})
Closest candidates are:
  f() at REPL[4]:1
  f(::Union{Int64, String}) at REPL[37]:1
Stacktrace:
 [1] top-level scope at REPL[40]:1
```

早速これを応用して先ほどのコードを書き直してみましょう。

```julia
julia> aslist(x::Int) = [x, ]
aslist (generic function with 4 methods)

julia> aslist(A::Union{Vector, Tuple}) = A
aslist (generic function with 5 methods)

julia> aslist(2)
1-element Array{Int64,1}:
 2

julia> aslist([1, 2, 3])
3-element Array{Int64,1}:
 1
 2
 3

julia> aslist((1, 2, 3))
(1, 2, 3)
```

これで同じ処理をまとめることができました。

さて、入力で`Int`, `Vector`, `Tuple`のみを想定する場合は問題なく動作するようになりましたが、複素数や多次元配列などの他の型の場合も想定したくなった場合はどのようにすればいいでしょうか。
最も簡単な解決策は、先ほどの`Union`型の定義部分に追加することです。

しかし、この方法を用いると、ある型がどのメソッドに割り当てられるのか、後からコードを読む人にとってわかりにくくなってしまいます。(Union型をいちいち見るのは目が痛くなります)
それを解決するために登場するのが`trait type`と、`trait function`です。

一度整理すると、この関数は現在2つの挙動を持っています。

- リストのような挙動
- それ以外の挙動

そこで、次のような挙動に対応した型を用意します。
これは*trait type*と呼ばれます。

```julia
julia> struct List end

julia> struct NoList end
```

さらに、これらの型のインスタンスを返す関数を書きます。
これは*trait function*と呼ばれます。
```julia
julia> islist(::Vector) = List()
islist (generic function with 1 method)

julia> islist(::Tuple) = List()
islist (generic function with 2 methods)

julia> islist(::Int) = Nonlist()
islist (generic function with 3 methods)
```

すると問題の関数は以下のように書けます。

```julia
julia> aslist(x) = aslist(islist(x), x)
aslist (generic function with 5 methods)

julia> aslist(::List, x) = x
aslist (generic function with 6 methods)

julia> aslist(::NoList, x) = [x, ]
aslist (generic function with 7 methods)
```

少し複雑になってきたので整理します。

まず、
`aslist(x)`が呼ばれると、これは1引数なので、`aslist(x) = aslist(islist(x), x)`が呼ばれます。

そして`islist(x)`の返り値によって、`aslist(::List, x) = x`もしくは`aslist(::NoList, x) = [x, ]`が呼ばれます。
ちなみに、`aslist(::List, x) = x`と`aslist(::NoList, x) = [x, ]`の第一引数は、その型のみが必要であり、その値は必要ないので、変数名が省略されています。

さて、このような書き方を用い、trait functionを追加することで、"簡単に"、"可読性を保ったまま"、この関数を他の型にも拡張することができます。

また、このような書き方はJuliaコンパイラによる強力な最適化が働き、ほとんどオーバーヘッドを気にせず使えます。(詳しくは元記事を参照してください！)

さて、前置きが長くなりましたが、実際にcombine_axisを確認していきましょう。

