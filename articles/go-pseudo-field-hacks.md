---
title: "Go コンパイラを制御する疑似フィールドハック集"
emoji: "🪄"
type: "tech"
topics: ["go", "compiler", "govet", "struct", "generics"]
published: true
publication_name: "yumemi_inc"
---

:::message
Go ではタブインデントが推奨されていますが，この記事では Web ブラウザ上での見やすさに配慮して半角スペースを採用しています。
:::

# はじめに

社内の Slack で，同僚がこんな話をしていました。

***💬「フィールドが約 1,250 個ある struct のせいで， `-race` 付きのコンパイルが 476 秒かかるようになった」***

コードジェネレータが作った，データベースのスキーマを表す struct です。テーブルと列を全部入れ子にした巨大な struct を 1 つのグローバル変数に入れていたところ，コンパイラが **その struct 用の等価比較関数** を律儀に生成していて，その関数が巨大になっていました。

最終的な対策は，列を表す型にたった 1 行を足すことでした。

```go
type Col[T any] struct {
    _ [0]func() // ← これ
    // ...
}
```

これで 476 秒が 39 秒になったそうです。

***💬「ハックすぎるだろこれ」***

と思って調べてみたら， `log/slog` の `Value` も同じことをしていました。こういう **「中身のない `_` フィールドを置いて，コンパイラや `go vet` の振る舞いを変える」** テクニックは，標準ライブラリの中にけっこうあります。せっかくなので， AI に思いつく限り挙げてもらったものを，実際にビルドして確かめながらまとめてみました。

:::message
記事中の実行結果は，すべて Go 1.27.3（darwin/arm64）で確認したものです。
:::

# 一覧

先に一覧を置いておきます。

| フィールド                    | 効果                       | 誰が見るか      | 実例                                     |
|:-------------------------|:-------------------------|:-----------|:---------------------------------------|
| `_ [0]func()`            | `==` での比較を禁止する          | コンパイラ      | `slog.Value`                           |
| `_ [0]*T`                | 型パラメータ違いの型への変換を禁止する     | コンパイラ      | `atomic.Pointer[T]`<br>`weak.Pointer[T]` |
| `_ struct{}`             | 別パッケージからのキーなしリテラルを禁止する | コンパイラ      | protobuf                               |
| `_ noCopy`<br>`_ [0]sync.Mutex` | 値のコピーを警告する              | `go vet`   | `sync.WaitGroup`<br>`atomic.Int64` など  |
| `_ structs.HostLayout`   | C と同じメモリレイアウトにすることを宣言する  | コンパイラ      | Go 1.23 で追加                           |
| `_ byte`                 | ポインタのアドレスが重ならないようにする     | ランタイム      | 拙作の `feature`                          |

どれも **`_` という名前のフィールドなので，コードからは読み書きできません。** 型の性質を変えるためだけに置くフィールドです。 `[0]T` のように長さ 0 の配列を使うものは **サイズも 0** なので，メモリ上のコストもありません（ただし置く場所に注意が必要です。後述します）。

# `_ [0]func()`: `==` での比較を禁止する

## 何が起こるか

Go の関数型は比較できません。そして，比較できない型をフィールドに持つ struct も比較できなくなります。長さ 0 の配列 `[0]func()` も，要素の型が比較できないので比較できない型です。

```go
type Value struct {
    _   [0]func()
    num uint64
}

func main() {
    a, b := Value{}, Value{}
    _ = a == b // コンパイルエラー: invalid operation: a == b (struct containing [0]func() cannot be compared)

    m := map[Value]int{} // これもコンパイルエラー: invalid map key type Value
}
```

`log/slog` の `Value` は，まさにこの目的で使っています。

> ```go
> type Value struct {
>     _ [0]func() // disallow ==
>     // ...
> }
> ```

`slog.Value` は中身に `any` を持っているので， `==` で比べると **正しく比較できなかったり， panic したりする** ことがあります。比較には `Value.Equal()` を使ってほしいので， `==` を最初から使えないようにしているわけです。

## コンパイラが等価比較関数を作らなくなる

冒頭の話は，この副作用の方です。

比較できる struct に対して，コンパイラは **等価比較関数** を自動で生成します。 `==` で比べたり， `map` のキーや `interface` に入れて比べたりするときに使われる関数です。フィールドが多ければ，そのぶん関数も大きくなります。

比較できない struct には，この関数がそもそも要りません。末端の型に `_ [0]func()` を入れると，それを含む外側の struct も全部比較できない型になるので， **入れ子の struct 全体で等価比較関数が生成されなくなります。**

実際に再現してみました。冒頭の話と同じように，ジェネリクスの列の型 `Col[T]` を並べたテーブル struct を，さらに入れ子にして 1 つのグローバル変数にまとめたコードを生成します。

```go
type Col[T any] struct {
    // _ [0]func() ← これの有無で比べる
    Table *Table
    Name  string
}

type t0_0 struct {
    Table
    C0  Col[int64]
    C1  Col[string]
    // ...（1 テーブルあたり 11 列）
}

// ...

var Tbl = struct {
    g0
    g1
    // ...
}{}
```

列の数を変えながら， `go build -race` でこのパッケージだけをコンパイルし直す時間を測りました。

| 列の数   | 等価比較関数の最大サイズ | `_ [0]func()` なし | `_ [0]func()` あり |
|:------|:-------------|:----------------|:----------------|
| 990   | 約 700 KB     | 2.40 秒          | 0.58 秒          |
| 1,980 | 約 1.4 MB     | **13.10 秒**     | **0.48 秒**      |

`_ [0]func()` がない場合は，トップレベルの struct 用に **1 MB を超える等価比較関数が 1 本** 生成されていました。列が倍になると，コンパイル時間は 5 倍以上に伸びています。 `_ [0]func()` を入れると，等価比較関数は 1 本も生成されなくなり，時間も列の数にほぼ関係なくなりました。

同僚のプロジェクトの struct は手元の再現コードより複雑だったので，そのぶん差が大きく出たのでしょう。

:::message
等価比較関数は `go build -gcflags=-S` で出力されるアセンブリの中に `type:.eq.〜` という名前で見つかります。今回はこのサイズを集計しました。
:::

:::message alert
**比較したい型に入れてはいけません。** 外側の struct まで全部比較できなくなります。 `==` で比べたり `map` のキーにしたりする必要がない型，たとえばスキーマ定義やレジストリのような型にだけ使ってください。
:::

## ポインタにするのではダメなのか

***💬「入れ子の struct を適宜ポインタにすれば，比較はアドレスだけになるのでは？」***

私も最初そう思いました。同僚がこれも試していましたが，グループの階層をポインタにする方法では， 39 秒に対して 53 秒止まりだったそうです。ポインタの先の struct もまだ比較できる型なので，その型の等価比較関数は生成され続けます。巨大な関数 1 本が数本に分かれるだけです。

***💬「でも，どこにも `==` なんて書いてないのに，なんで生成されるの？」***

等価比較関数は，実行時に使う型の情報（型記述子）の一部だからです。 `any` に入れた値どうしを `==` で比べるとき，ランタイムは中身の型の型記述子から等価比較関数を取り出して呼びます。

```go
var a, b any = tblItem{}, tblItem{}
_ = a == b // コンパイル時には中身の型が分からないので，実行時に tblItem の等価比較関数を呼ぶ
```

コンパイラには，その型の値がどこかで `any` に入れられて比べられるかどうかが分かりません。なので， **比較できる型の型記述子を出力するときは，等価比較関数も一緒に生成します。** `*tblItem` の型記述子は指す先の `tblItem` の型記述子を参照しているので，ポインタにしても `tblItem` の等価比較関数は生成されます。

ポインタは「共有したい」「 `nil` で無いことを表したい」といった理由で使うものです。コンパイラの生成物を減らしたいなら，比較できない型にする方が素直です。

# `_ [0]*T`: 型パラメータ違いの型への変換を禁止する

Go では，フィールドの構成が同じ struct どうしは型変換できます。これはジェネリクスの型でも同じです。

```go
type Plain[T any] struct{ v int }

// T を一切使っていないので，中身はどちらも struct{ v int }
_ = Plain[string](Plain[int]{}) // ← コンパイルが通ってしまう
```

型パラメータ `T` をフィールドで一度も使っていないと， `Plain[int]` と `Plain[string]` の中身は同じになり，変換できてしまいます。 `T` を中身の型として扱いたいのに，これは困りますね。

そこで `_ [0]*T` を置いて，フィールドの構成に `T` を含めます。

```go
type Phantom[T any] struct {
    _ [0]*T
    v int
}

_ = Phantom[string](Phantom[int]{})
// コンパイルエラー: cannot convert Phantom[int]{} (value of struct type Phantom[int]) to type Phantom[string]
```

`sync/atomic` の `Pointer[T]` がこれを使っています。 `unsafe.Pointer` しか持っていないので，何もしないと変換できてしまうためです。

> ```go
> type Pointer[T any] struct {
>     // Mention *T in a field to disallow conversion between Pointer types.
>     // See go.dev/issue/56603 for more details.
>     // Use *T, not T, to avoid spurious recursive type definition errors.
>     _ [0]*T
>
>     _ noCopy
>     v unsafe.Pointer
> }
> ```

`T` ではなく `*T` にしているのは，コメントにあるとおり，再帰的な型定義のエラーを避けるためです。 `weak.Pointer[T]` も同じ書き方をしています。

https://github.com/golang/go/issues/56603

# `_ struct{}`: 別パッケージからのキーなしリテラルを禁止する

struct のリテラルは，フィールド名を省略して順番で書くこともできます。

```go
p := other.P{1, 2} // キーなし
p := other.P{X: 1, Y: 2} // キーあり
```

キーなしで書かれると，ライブラリ側でフィールドを追加したり並べ替えたりしたときに，利用者のコードが壊れたり，別のフィールドに値が入ったりします。そこで `_ struct{}` を置きます。

```go
package other

type P struct {
    _    struct{}
    X, Y int
}
```

```go
package main

_ = other.P{X: 1, Y: 2}         // OK
_ = other.P{struct{}{}, 1, 2}   // コンパイルエラー: implicit assignment to unexported field _ in struct literal of type other.P
```

`_` は大文字で始まらないので，非公開のフィールドとして扱われます。キーなしリテラルは全フィールドに値を書く必要があるので，非公開のフィールドがあると別パッケージからは書けなくなる，という仕組みです。

:::message
**同じパッケージの中からは，キーなしでも書けてしまいます。** `other` パッケージ内で `P{struct{}{}, 1, 2}` と書くとコンパイルが通りました。あくまで利用者向けの制約です。
:::

# `_ noCopy`: 値のコピーを警告する

`sync.Mutex` のように，コピーすると壊れる型があります。 `go vet` の `copylocks` は，こういう型の値のコピーを検出してくれます。

`copylocks` が見ているのは **「ポインタレシーバで `Lock()` と `Unlock()` を持つ型かどうか」** です。なので，何もしない `Lock()` と `Unlock()` を持つ空 struct を置けば，自分の型もコピー禁止にできます。

```go
type noCopy struct{}

func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}

type A struct {
    _ noCopy
    N int
}

func useA(a A) {} // 値で受け取っている
```

```console
$ go vet ./...
vet/main.go:20:13: useA passes lock by value: hacks/vet.A contains hacks/vet.noCopy
vet/main.go:26:10: call of useA copies lock value: hacks/vet.A contains hacks/vet.noCopy
```

見た目はかなりハックですが，これも標準ライブラリで堂々と使われています。 `sync.WaitGroup` ， `sync.Cond` ， `atomic.Int64` などがそうです。

> ```go
> // Lock is a no-op used by -copylocks checker from `go vet`.
> func (*noCopy) Lock()   {}
> func (*noCopy) Unlock() {}
> ```

自分で型を定義するのが面倒なら， `_ [0]sync.Mutex` でも同じ警告が出ます。長さ 0 の配列なのでサイズは 0 です。

```console
vet/main.go:21:13: useB passes lock by value: hacks/vet.B contains sync.Mutex
```

:::message alert
**これはコンパイラではなく `go vet` の機能です。** `go vet` を CI などで回していなければ，コピーしてもそのまま通ります。
:::

# `_ structs.HostLayout`: C と同じメモリレイアウトにすることを宣言する

Go 1.23 で追加された，公式の疑似フィールドです。

```go
import "structs"

type Point struct {
    _    structs.HostLayout
    X, Y int32
}
```

Go の言語仕様は，struct のフィールドがメモリ上でどう並ぶかを保証していません。C の関数やシステムコールに struct を渡すときは，C と同じ並びになっていないと困ります。 `structs.HostLayout` を置くと，その struct は **ホストのプラットフォームの C ABI に従って並べる** という意味になります。

Proposal の議論によると，今の Go のレイアウトはどのプラットフォームでもホストのレイアウトと一致しているので，現時点で置いても挙動は変わりません。将来コンパイラがレイアウトを最適化するようになっても，この struct は対象外にしてね，という目印です。

https://github.com/golang/go/issues/66408

ドキュメントには「名前は `_` にして，struct の先頭に置く」という慣習も書かれています。

# `_ byte`: ポインタのアドレスが重ならないようにする

これは逆向きのハックで，「サイズを 0 にしない」ためのフィールドです。

```go
type Opaque struct{}
type OpaqueByte struct{ _ byte }

fmt.Printf("%p %p\n", new(Opaque), new(Opaque))
// 0x100e701e0 0x100e701e0  ← 同じアドレス

fmt.Printf("%p %p\n", new(OpaqueByte), new(OpaqueByte))
// 0x304edf73c0f8 0x304edf73c0f9  ← 別のアドレス
```

サイズ 0 の値は同じアドレスに割り当てられることがあります。 **ポインタを一意な識別子として使いたいときは，サイズ 0 の型では困る** ので， `_ byte` で 1 バイト持たせて別々のアドレスにします。

以前の記事で紹介した拙作のライブラリでは，これをコンテキストキーの識別子に使っています。

https://zenn.dev/mpyw/articles/go-context-feature-flags

# 標準ライブラリの中でしか使えないもの

以下の 2 つはコンパイラが特別扱いしているもので，同じ型を自分のパッケージで定義しても，コンパイラは特別扱いしてくれません。

| フィールド              | 効果                         | 実例                    |
|:-------------------|:---------------------------|:----------------------|
| `_ align64`        | 32 bit 環境でも 64 bit 境界に揃える | `atomic.Int64` など     |
| `_ sys.NotInHeap`  | ヒープに割り当てない型であることを示す        | ランタイム内部               |

> ```go
> // align64 may be added to structs that must be 64-bit aligned.
> // This struct is recognized by a special case in the compiler
> // and will not work if copied to any other package.
> type align64 struct{}
> ```

ただし `align64` は，それを含む `atomic.Int64` を長さ 0 の配列にすれば間接的に使えます。 32 bit 環境の 386 と arm でコンパイルして，フィールドのオフセットを確認しました。

```go
type Plain struct {
    A int32
    B int64 // オフセット 4（ 32 bit 環境では 4 バイト境界）
}

type Aligned struct {
    A int32
    B struct {
        _ [0]atomic.Int64
        v int64
    } // オフセット 8（ 8 バイト境界に揃う）
}
```

32 bit 環境で `int64` のアトミック操作をするときは， 8 バイト境界に揃っていないと正しく動きません。素直に `atomic.Int64` 型を使えば済む話ですが，こういう書き方もできます。

# 置く場所に注意: サイズ 0 のフィールドは先頭に

サイズ 0 のフィールドは， **struct の最後に置くとサイズが増えます。**

```go
type Head struct {
    _ [0]func()
    X int64
}

type Tail struct {
    X int64
    _ [0]func()
}

fmt.Println(unsafe.Sizeof(Head{}), unsafe.Sizeof(Tail{}))
// 8 16
```

最後のフィールドのサイズが 0 だと，そのフィールドのアドレスが struct の外を指してしまいます。それを避けるために，コンパイラが末尾にパディングを入れるためです。 `slog.Value` も `atomic.Pointer` も，疑似フィールドを先頭に置いていますね。

# ハックを取りまとめたユーティリティパッケージ

Protocol Buffers の Go 実装には，この手の型をまとめた `pragma` パッケージがあります。

```go
package pragma

type NoUnkeyedLiterals struct{}

type DoNotImplement interface{ ProtoInternal(DoNotImplement) }

type DoNotCompare [0]func()

type DoNotCopy [0]sync.Mutex
```

`internal` なので外からは使えませんが，名前が分かりやすいので，自分のプロジェクトで同じような型を定義するときの参考になります。 `DoNotImplement` は疑似フィールドではありませんが， interface に非公開の型を引数に取るメソッドを入れて，外部のパッケージから実装できないようにするテクニックです。

https://github.com/protocolbuffers/protobuf-go/blob/master/internal/pragma/pragma.go

# まとめ

- **`_ [0]func()` で比較を禁止できる。** 巨大な struct で等価比較関数が生成されるのも防げるので，コンパイルが遅いときは疑ってみよう
- **`_ [0]*T` で，型パラメータ違いの型への変換を禁止できる**
- **`_ struct{}` で，別パッケージからのキーなしリテラルを禁止できる**
- **`_ noCopy` や `_ [0]sync.Mutex` で，コピーを `go vet` に警告させられる**
- **`_ structs.HostLayout` で， C と同じメモリレイアウトにすることを宣言できる**
- **`_ byte` で，ポインタのアドレスが重ならないようにできる**
- **サイズ 0 の疑似フィールドは先頭に置こう！**

普段こんなことを意識しながら Go を書くことはないと思いますが，困ったときに思い出してもらえたら嬉しいです。
