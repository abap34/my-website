---
toc: true
layout: post
description: Juliaの型変換とpromotion(昇格)について解説します。
categories: [JuliaLang]
title: Juliaの型変換とpromotion(昇格)
image: images/promote.png
---



先日, 

{% twitter https://twitter.com/abap34/status/1371651401311424513 %}

{% twitter https://twitter.com/abap34/status/1371651936089337859 %}

 

というようなツイートをしたところ



{% twitter https://twitter.com/MathSorcerer/status/1371652683006799876 %}

{% twitter https://twitter.com/LirimyDh/status/1371653754215223302 %}

というpromotionを使った解決策を教えていただきました。

実際に簡潔な実装にできるかどうかは置いておいて、昇格や型変換に興味を持ったので、

Juliaのドキュメントのconversion and promotionという章の日本語訳を元にしてこれらを解説し、またこれらを使った実装を示します。

(以下、日本語訳。意訳・省略・補足等を多用しています。完全な翻訳はhttps://mnru.github.io/julia-doc-ja-v1.0/manual/conversion-and-promotion.html#conversion-and-promotion-1などを参照してください。)

# [Conversion and Promotion](https://docs.julialang.org/en/v1/manual/conversion-and-promotion/#conversion-and-promotion)

Juliaは、数学的な演算子の引数を共通の型に昇格するシステム(暗黙の型変換)を備えています。

このシステムについて、ドキュメントでは「整数と浮動小数点数」、「数学的演算子と初等関数」、「型」、「メソッド」他のいくつかの章で説明されていますが、

この章では、昇格システムがどのように働くのか、またこのシステムをどのように新しい型に拡張するのか、組み込みの数学的な演算子以外に適用するのかを説明します。

(注:のちに説明されますが、この"昇格"は型システムにおける親タイプなどとは関係なく、単にpromotionの訳語として用いています)

## 従来のプログラミング言語

従来のプログラミング言語の昇格システムは、主に2つの陣営に分類することができます。

### 1: 暗黙の型変換を行う言語

ほとんどの言語では、組み込みの数値型は、演算子(+, -, *など)の被演算子として用いられた時、期待される結果が得られるように、自動的に共通の型に昇格します。

例えば、C, Java, Perl, Pythonなどはこのような部類に属します。

このような言語で 1 + 1.5を計算することを考えてみます。

このような式を書くときに、このことを意識する人はほとんどいませんが、整数と浮動小数点はそのままでは加算できないので、コンパイラやインタプリタは加算の前に変換を行う必要があります。

つまりこの場合は 1 + 1.5は、1.0 + 1.5と変換され、2.5という結果が得られます。

このような自動変換のための複雑なルールは、必然的にこれらの言語の仕様や実装に含まれることになります。

### 2:暗黙の型変換を行わない言語

この陣営はAdaやMLなど非常に"厳格"な静的型付けの言語が属しています。

このような言語では、全ての変換はプログラマから明示的に指定される必要があります。したがって、式`1 + 1.5`はAdaとMLのいずれでもコンパイルエラーとなります。

代わりに、`real(1) + 1.5`と書いて、加算を処理する前に明示的に整数である1を浮動小数点に変換する必要があります。

しかし、常に明示的な変換をする必要があるのは不便なので、例えばAdaでさえも、整数リテラルは自動で期待される整数型に昇格し、また浮動小数のリテラルも自動で同じように適切な浮動小数の型に昇格します。





#### Juliaの暗黙の型変換

**ある意味では**、Juliaは「暗黙の型変換を行わない」陣営に属します。

Juliaの数学演算子は特別な構文(訳注:中置構文)を持つ関数にすぎず、演算子に限って特殊な処理が行われることで、**被演算子がが自動的に変換**されることはないからです。

しかし、数学的な演算子を多様な型が混在する引数に適用する、という状況は、多重ディスパッチの一つの例に過ぎません。

そのため、Juliaのシステムを暗黙の型変換のような機構に生かすことができます。

#### Juliaの異なる型同士の演算の実装

Juliaは、ある被演算子の型の組み合わせに関して、特定の実装が存在しない場合呼び出されるルールがあります。

このルールは全ての被演算子の種類の組み合わせをカバーしていて、これによって被演算子を共通の型に昇格させる処理を行うことができます。

従って、これを用いて同じ型に変換した被演算子に対して、演算子を呼び出します。

また、ユーザー定義の型については、

- 他の型との変換方法を定義し、
- 他の型と混合したときにどの型に昇格するかを定義する

ことで、簡単にこのシステムを利用することができます。

これについては、のちのpromotionの項目で説明します。

## Conversion

ある型Tの値を取得する標準的な方法は、その型のコンストラクタ`T(x)`を呼び出すことです。

しかし、プログラマーの明確な要求を必要とせず、値を他の型から変換したほうが便利なケースもあります。

例えば、値を配列に代入するときです。

`Vector{Float64}`の配列Aがあったとき, `A[1] = 2`という式は、2を自動的に`Int`から`Float64`に変換されることで動作します。

これは`convert`関数によって行われます。

#### convert関数

`convert`関数は通常, 2つの引数を取ります。

第一引数は型オブジェクト、第二引数はその型に変換する値です。

返される値は、与えられた型のインスタンスに変換された値です。

実際の動作を確認してみます。



```julia
julia> x = 12
12

julia> typeof(x)
Int64

julia> convert(UInt8, x)
0x0c

julia> typeof(ans)
UInt8

julia> convert(AbstractFloat, x)
12.0

julia> typeof(ans)
Float64

julia> a = Any[1 2 3; 4 5 6]
2×3 Array{Any,2}:
 1  2  3
 4  5  6

julia> convert(Array{Float64}, a)
2×3 Array{Float64,2}:
 1.0  2.0  3.0
 4.0  5.0  6.0
```

この時、変換が常に行えるとは限りません。

要求された変換が未知の場合は、`convert`は`MethodError`を返します。

```julia
julia> convert(AbstractFloat, "foo")
ERROR: MethodError: Cannot `convert` an object of type String to an object of type AbstractFloat
[...]
```

また、いくつかの言語では、文字列を数値と見做したり、数値を文字列としてフォーマットすることを変換と見なすことがあります。(多くの動的言語では自動でこれが行われます)

例えばPythonでは、次のような処理を行うことができます。

```python	
Python 3.7.2 (v3.7.2:9a3ffc0492, Dec 24 2018, 02:44:43)
[Clang 6.0 (clang-600.0.57)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> s = "123"
>>> int(s)
123
>>> type(_)
<class 'int'>
```



しかし、Juliaではそのようなことはありません。いくつかの文字列が数値と見なすことができても、ほとんどの文字列は数値の表現として適切ではなく、文字列全体の集合のうち、非常に限られた部分集合のみが数値の表現として有効です。

ですから、

```julia
julia> s = "123"
"123"

julia> convert(Int, s)
ERROR: MethodError: Cannot `convert` an object of type String to an object of type Int64
Closest candidates are:
  convert(::Type{T}, ::T) where T<:Number at number.jl:6
  convert(::Type{T}, ::Number) where T<:Number at number.jl:7
  convert(::Type{T}, ::Ptr) where T<:Integer at pointer.jl:23
  ...
Stacktrace:
 [1] top-level scope at REPL[2]:1
```

となります。

そのためJuliaでは、専用の関数`parse`を使って明示的にこの操作をする必要があります。

```julia
julia> parse(Int, s)
123

julia> typeof(ans)
Int64
```



### いつconvertは呼び出されるのか

`convert`は次のような状況で呼び出されます。

- 配列に代入するとき(要素の型に)
- オブジェクトのフィールドに代入するとき(そのフィールドの宣言された型に)
- オブジェクトを`new`で構築するとき(宣言されたフィールドの型に).   
  (注:`new`は内部コンストラクタメソッドでアクセスできる特殊な関数で、これを用いることで構造体を生成できます。)
- 変数の宣言時に型を指定したとき(その型に)
  (例:`local x. :: T`)
- 帰り値の型が宣言されている関数が、値を返すとき(その型に)
- `ccall`に値を渡すとき(対応する引数の型に)

### Conversion vs. Construction

さて、`convert(T, x)`の動作は、`T(x)`とほぼ同じように見えます。

実際、通常はそうですが、意味合いは大きく異なります。

まず、`convert`は多くの場合、暗黙のうちに呼び出されるので、実行するケースは安全かつ特殊な動作が予見されないときに限られます。

また、`convert`は基本的に同じ種類のものを表す型の間でのみ変換します。(例えば、同じ数値の異なる表現など。)

以下に、コンストラクタと`convert`が異なるケースを、4種類ほど示します。

#### 引数に関係ない型のコンストラクタ

コンストラクタの中には、「変換」という概念を表さないものもあります。

例えば、2秒のタイマーを生成するコード、`Timer(2)`は整数をタイマーに"変換"しているわけではありません。

#### mutableなコレクション

`convert(T, x)`は、`x`がすでに`T`であれば、元の`x`を返すように動作します。

しかし、Tがmutableなコレクションであるとき、`x`から要素をコピーして,` T(x)`は常に新しいコレクションを作ります。

実際に確かめてみると、確かに`convert`は同じオブジェクトを返していますが、`T(x)`は異なるオブジェクトを返していることがわかります。

```julia
julia> objectid(A)
0x6fc8a9e14ca98c31

julia> objectid(convert(Array{Int64}, A))
0x6fc8a9e14ca98c31

julia> objectid(Array{Int64}(A))
0xbff8577572344790
```



#### ラッパー型

値を"ラップする"ような型では、コンストラクタは引数がすでにその型であったとしても、新しいオブジェクトでラップします。例えば、`Some(x)`という値が存在するかどうかを(結果が`Some`か`nothing`かで)示す型があったとします。

このとき, `x = Some(y)`であるような`x`であろうと、`Some(x)`は、二重にラップした結果、つまり`Some(Some(y))`を返します。しかし、`convert(Some, x)`は単に`x`を返します。なぜなら、`x`はすでに`Some`であったからです。

#### 自身の型を返さないコンストラクタ

非常に稀ですが、`T(x)`が型`T`の値を返さないことがあります。

(実際の実装は異なりますが、例えば`Transpose(Transpose(x)) == x`というような対合性がをもつ状況など)

このような時でも、`convert(T, x)`は常に型`T`の値を返しますから、結果が異なるようになります。

### 新しい変換を定義する

新しい型を定義するとき、最初はその型を生成する方法をコンストラクタとして定義すべきです。

暗黙の変換が有用であることが明らかになり、コンストラクタが「安全に」使用できる時は`convert`を追加しても問題ありません。

これらのメソッドは、適切なコンストラクタを呼び出すだけなので、通常は非常に単純で、以下のようになります。

```julia
convert(::Type{MyType}, x) = MyType(x)
```

第一引数はシングルトン型の`Type{MyType}`です。

シングルトン型は抽象型の一種で、`Type{T}`のインスタンスは、オブジェクト`T`のみです。従って、この関数は、第一引数が`MyType`の場合のみ動作します。

従って、第一引数の値は明らかなので、引数を受け取る必要はなく、引数名が省略された記法が用いられています。

また、これを用いてすでに引数が型`T`である時の処理も実装できます。

```julia
convert(::Type{T}, x::T) where {T<:Number} = x
```

# Promotion

昇格(Promotion)とは、さまざまな型の値を共通の型に変換することを指します。

厳密には成り立たないこともありますが、一般的には、値が変換される共通の型が元の値のすべてを忠実に表現することが想定されています。

つまり、値が「より大きな」つまり入力された全ての値を単一の共通型で表現できる型に変換されます。

これが、「昇格(Promotion)」という言葉が用いられる由来です。

しかし、これをオブジェクト思考における(構造的な)親タイプやJuliaの型システムにおける抽象型の親タイプの概念と混同しないよう注意してください。

昇格は型の階層とは関係なく、表現の変換にのみ関連する概念です。例えば、全ての`Int32`の値は、実質的には`Float64`での表現に変換できますが、`Int32`は`Float64`のサブタイプではありません。

さて、Juliaは「より大きな」型に昇格させるため、`promote`関数を用います。

これは、任意の引数をとって、共通の型に変換されたタプルを返すような関数です。

`promote`は、もしも昇格が不可能な場合は例外を発生させます。

この最も一般的な使用例は、数値の引数を昇格させる時です。

```julia
julia> promote(1, 2.5)
(1.0, 2.5)

julia> promote(1, 2.5, 3)
(1.0, 2.5, 3.0)

julia> promote(2, 3//4)
(2//1, 3//4)

julia> promote(1, 2.5, 3, 3//4)
(1.0, 2.5, 3.0, 0.75)

julia> promote(1.5, im)
(1.5 + 0.0im, 0.0 + 1.0im)

julia> promote(1 + 2im, 3//4)
(1//1 + 2//1*im, 3//4 + 0//1*im 
```

- 浮動小数点数は、引数の浮動小数点数型の中で最大のものに昇格します。

- 整数はネイティブマシンのワードサイズと引数の整数型の最大値のどちらか大きい方に昇格します。

- 整数と浮動小数点数がいずれも引数に含まれている場合、全ての値を格納できる大きさの浮動小数点数に昇格します。

- 整数と有理数の場合は有理数に、有理数と浮動小数点数の場合は、浮動小数点数に昇格します。
- 実数値と複素数値が混ざった場合は、適切な種類の複素数型に昇格します。



# Promotion Rulesの定義

さて、この昇格を制御するにあたっては、原理的には`promote`関数を直接定義することでも実現できますが、

その場合多くの引数の型に対応した冗長な定義が必要になります。

そのため、`promote`の定義は`promote_rule`という補助的な関数で定義することが可能となっています。

`promote_rule`関数は、型オブジェクトのペアを受け取り、それら二つの型が与えられた時に昇格する型を返すような関数です。

```julia
promote_rule(::Type{Float64}, ::Type{Float32}) = Float64
```

また、返す型オブジェクトは、必ずしも引数の型に含まれている必要はありません。

例えば、以下に示す昇格ルールは、どちらもJuliaの標準ライブラリに存在する定義です。

```julia
promote_rule(::Type{UInt8}, ::Type{Int8}) = Int
promote_rule(::Type{BigInt}, ::Type{Int8}) = BigInt
```

また、`promote_rule(::Type{A}, ::Type{B})` と `promote_rule(::Type{B}, ::Type{A}`の両方を定義する必要がないことに注意してください。(`promote`は対称性を前提として実装されています)

実際の実装は[promotion.jl](https://github.com/JuliaLang/julia/blob/master/base/promotion.jl)で見ることができますが、簡潔に処理の流れを記すと、

1. ユーザーによって(もしくはpromotion.jl内で)定義された`promote_rule`を用いて、`promote_type`関数が昇格させる型を返す(この際対称性の処理が行われる)
2. `promote_type`が返した値に基づいて`_promote`で値がconvert(昇格)される
3. `promote`でエラーチェック(きちんと昇格されているかチェック)が行われて値を返す

関連する処理のみを抜き出したものを掲載します。(簡単のため、 異なる型の2引数の場合のみです)

```julia
# @_inline_metaはどうやら@inlineと同じ働きをするようです(確信はないので、詳しい方は教えてくださると嬉しいです)

# 1
function promote_type(::Type{T}, ::Type{S}) where {T,S}
    @_inline_meta
    # Try promote_rule in both orders. Typically only one is defined,
    # and there is a fallback returning Bottom below, so the common case is
    #   promote_type(T, S) =>
    #   promote_result(T, S, result, Bottom) =>
    #   typejoin(result, Bottom) => result
    promote_result(T, S, promote_rule(T,S), promote_rule(S,T))
end


#2
function _promote(x::T, y::S) where {T,S}
    @_inline_meta
    R = promote_type(T, S)
    return (convert(R, x), convert(R, y))
end

#3
function promote(x, y)
    @_inline_meta
    px, py = _promote(x, y)
    not_sametype((x,y), (px,py))
    px, py
end

not_sametype(x::T, y::T) where {T} = sametype_error(x)

not_sametype(x, y) = nothing

function sametype_error(input)
    @_noinline_meta
    error("promotion of types ",
          join(map(x->string(typeof(x)), input), ", ", " and "),
          " failed to change any arguments")
end
```



# 実装

さて、Juliaにおけるpromotionを完全に理解したところで、今回のケースで実際に使ってみようと思います。

`variable`は実数との演算を行うわけですから,設定すべき`promote_rule`は、

```julia
Base.promote_rule(::Type{<:Real}, ::Type{Variable}) = Variable
```

であり、これらのconvert方法を用意してやればいいです。(すでに内部コンストラクタメソッドが定義されています。)

```julia
Base.convert(::Type{Variable}, x::T) where {T <: Real} = Variable(x)
```

こうすることで、今まで

```julia
Base.:+(x1::Variable, x2::Variable) = _Add(x1, x2)
Base.:+(x1::Variable, x2::T) where {T <: Real} = _Add(x1, Variable(x2))
Base.:+(x1::T, x2::Variable) where {T <: Real} = _Add(Variable(x1), x2)
```

となっていたコードは、

```julia
Base.:+(x1::Variable, x2::Variable) = _Add(x1, x2)
Base.:+(x1::Variable, x2::T) where {T <: Real} = +(promote(x1, x2)...)
Base.:+(x1::T, x2::Variable) where {T <: Real} = +(promote(x1, x2)...)
```

となりました。


.....あれ?


(promotionを使ったもっといい書き方がある気がする....)



