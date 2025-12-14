---
title: "Feature Flag の量産に耐えられる Go のライブラリを作りました"
emoji: "🚩"
type: "tech"
topics: ["go", "context", "featureflags", "oss", "generics"]
published: false
---

:::message
Go ではタブインデントが推奨されていますが，この記事では Web ブラウザ上での見やすさに配慮して半角スペースを採用しています。
:::

:::message
この記事は 7 割ぐらい AI が書いています。
:::

# ご挨拶

この記事は [Go - Qiita Advent Calendar 2025](https://qiita.com/advent-calendar/2025/go) (Series 2) の XX 日目の記事です。実は今回 2 回目の参加です。空いてる日があったので，ちょうどいい小ネタで埋めさせてもらいます。

↓ 前回の記事
https://zenn.dev/mpyw/articles/new-relic-go-agent-struggle

# 動機: コンテキストキーの構造体定義，冗長すぎない？

業務で Go プロジェクトを触っていて，特定のエンドポイントでだけ深いところにある共通ロジックの挙動を変えたいという場面がありました。俗に言う **Feature Flag**（機能フラグ）と呼ばれるものです。

***💬「よし， Go らしく [`context.Context`](https://pkg.go.dev/context) 経由で伝搬させよう」***

…と思ったものの，毎回こういうの書くの，正直しんどくないですか？

```go
type requestTimeoutKey struct{}

func WithRequestTimeout(ctx context.Context, timeout time.Duration) context.Context {
    return context.WithValue(ctx, requestTimeoutKey{}, timeout)
}
func GetRequestTimeout(ctx context.Context) (time.Duration, bool) {
    v, ok := ctx.Value(requestTimeoutKey{}).(time.Duration)
    return v, ok
}
```

1 つ 2 つならまだいい。でもフラグが増えてくると…

```go
type requestTimeoutKey struct{}
type coolFeatureXKey struct{}
type coolFeatureYKey struct{}

func WithRequestTimeout(ctx context.Context, timeout time.Duration) context.Context {
    return context.WithValue(ctx, requestTimeoutKey{}, timeout)
}
func GetRequestTimeout(ctx context.Context) (time.Duration, bool) {
    v, ok := ctx.Value(requestTimeoutKey{}).(time.Duration)
    return v, ok
}
func WithCoolFeatureX(ctx context.Context, enabled bool) context.Context {
    return context.WithValue(ctx, coolFeatureXKey{}, enabled)
}
func CoolFeatureXEnabled(ctx context.Context) bool {
    v, _ := ctx.Value(coolFeatureXKey{}).(bool)
    return v
}
func WithCoolFeatureY(ctx context.Context, enabled bool) context.Context {
    return context.WithValue(ctx, coolFeatureYKey{}, enabled)
}
func CoolFeatureYEnabled(ctx context.Context) bool {
    v, _ := ctx.Value(coolFeatureYKey{}).(bool)
    return v
}
// ...
```

もちろん，そんな急には増えないと思いますよ？ただ，こういうときは防御的な思想を持っておかないと，後で痛い目を見ることが多いです。こだわりすぎることによる納期の遅延や，仲間との関係性の悪化とか，そういうのが無い範囲であれば先手を打っておきたい，そう思いますよね？

でも，そのためだけにディレクトリ切りまくるのもなぁ…いやそうじゃないやろ，**もっと簡単に書きたいやろ**。

https://pkg.go.dev/github.com/mpyw/feature

勢いで作りました。パッケージ名は~~ググラビリティ低すぎる~~ `feature` です。無名なサードパーティのコンテキストライブラリにロックインされることは好ましくないので，

***💬「 Feature Flag だけに絞って使ってくれないかな〜」***

という願いを込めてこの名前にしました。~~これでもし治安を悪化させられても責任は取りません~~。

https://claude.ai/

https://oraios.github.io/serena/01-about/000_intro.html

https://go.dev/gopls/

こいつら組み合わせたら…もうエージェントの自走能力高すぎて，初期リリース版までは一瞬でできてしまった。もうシンギュラリティ来てるわ…

# Go 公式の Proposal と，なぜ Hold なのか

当初は Go 1.18 でジェネリクスが入ってからだいぶ経つし，「きっと誰かライブラリ作ってるだろ」と思って探したんですが，意外とドンピシャなものがない。

https://gofeatureflag.org/

Go でフィーチャーフラグといえばこういうライブラリも出てくるんですが，求めていたのが **「[`context.WithValue`](https://pkg.go.dev/context#WithValue) を型安全に使うためのユーティリティ」** だったので，ちょっと違いました。そんな仰々しいものは要らなかったんですよね。

とはいえ，実は Go 公式にも似たアイデアの Proposal は出ていたようです。ライブラリ作ったあとに気づいちゃったんですが…

https://github.com/golang/go/issues/49189

2021 年 10 月に [`@dsnet`](https://github.com/dsnet) 氏が提案したもので，まさに求めていたものなんですが，**3 年以上 Hold 状態のまま** です。

## Proposal の内容

[`@dsnet` 氏の提案](https://github.com/golang/go/issues/49189#issue-1037731122) は以下のような API です:

> ```go
> type Key[Value any] struct { name *string }
> 
> func NewKey[Value any](name string) Key[Value] {
>     return Key[Value]{&name}  // ← 引数のアドレスを使う
> }
> 
> func (k Key[V]) WithValue(ctx Context, val V) Context
> func (k Key[V]) Value(ctx Context) (V, bool)
> ```

> The Context API suffers from the lack of type safety since keys and values are both `interface{}`.

そう，それ！わかってるじゃん！

ここで注目すべきは `&name` の部分。`NewKey` の引数 `name` はローカル変数なので，**呼び出しごとに異なるアドレス**になります。同じ文字列 `"foo"` を渡しても，ポインタは別々。これで一意性を保証しています。

:::message
Go ではスタック変数だったものをコールスタック外に持ち出すようなコードを書いた場合， Go コンパイラがヒープ変数に昇格させてくれるために許される書き方です。

[Golangとメモリ管理について理解する](https://zenn.dev/tetuf22/articles/00e43b3221536d)

`go build -gcflags="-m" main.go` のように `-m` オプションを付けてビルドすると，どの変数がヒープに昇格したかを確認できます。
:::

私もこの案とほとんど同じ設計に結果的になっていたみたいですが，詳細面で異なる実装方式を取っています。詳しくは後述。

## 議論で出た代替案

Issue のコメント欄では別のアプローチも提案されています。

### [`@neild` 氏の「型をキーにする」案](https://github.com/golang/go/issues/49189#issuecomment-953367396)

> ```go
> // 型自体をキーとして使う
> func WithValueOf[T any](parent Context, val T) Context
> func ValueOf[T any](ctx Context) (T, bool)
> 
> // 使用例
> ctx = context.WithValueOf(ctx, myUser)
> user, ok := context.ValueOf[User](ctx)
> ```

シンプルで直感的ですが，**同じ型の値を複数持てない** という致命的な問題があります。

### [`@DeedleFake` 氏の「二重型パラメータ」案](https://github.com/golang/go/issues/49189#issuecomment-953229306)

> ```go
> // キー型と値型を分離
> type Key[K comparable, V any] struct{}
> 
> // 使用例
> type userKey struct{}
> var UserID = Key[userKey, int]{}
> ```

衝突は避けられますが，結局キー用の型定義が必要で，ボイラープレートが減りません。

## なぜ Hold なのか

Joe Tsai 氏（[`@dsnet`](https://github.com/dsnet)）は Go 開発チームのメンバーであり，実装の筋もいいです。実際 Issue を追っても，この案自体への技術的な反対意見は少ないことがわかります。 Hold になっている主な理由は **標準ライブラリへのジェネリクス導入方針が未確定** だからです。

1. **標準ライブラリのジェネリクス対応方針が決まっていない**
   - [Discussion #48287](https://github.com/golang/go/discussions/48287) で「既存 API をどうジェネリクス対応するか」が議論中
   - `context/v2` のような新パッケージを作るか，既存パッケージに追加するか，方針が定まっていない

2. **既存 API との共存問題**
   - [`context.WithValue`](https://pkg.go.dev/context#WithValue) / [`context.Value`](https://pkg.go.dev/context#Value) は残すのか， Deprecated にするのか
   - 移行パスをどう設計するか

3. **代替案との比較検討**
   - [`@neild` 案](https://github.com/golang/go/issues/49189#issuecomment-953367396) は「同じ型の値を複数持てない」という致命的問題がある
   - しかし Go チームは複数案を並行して検討しており，結論を急いでいない

要するに，**「この Proposal がダメ」というより「標準ライブラリ全体のジェネリクス対応方針が決まるまで待ち」** という状態です。

## 各アプローチの比較

各アプローチを比較してみましょう。

| アプローチ                                                                                   | コンセプト          | API サンプル                                              | 懸念点          |
|:----------------------------------------------------------------------------------------|:---------------|:------------------------------------------------------|:-------------|
| **従来の空構造体**                                                                             | 型でキーを区別        | `type myKey struct{}`<br>`ctx.Value(myKey{}).(int)`   | 全てが冗長        |
| **[`@dsnet` 案](https://github.com/golang/go/issues/49189#issue-1037731122)**            | ポインタで<br>キーを区別 | `key := context.NewKey[int]("k")`<br>`key.Value(ctx)` | （後述）         |
| **[`@neild` 案](https://github.com/golang/go/issues/49189#issuecomment-953367396)**      | 型自体がキー         | `context.ValueOf[User](ctx)`                          | 同じ型の値を複数持てない |
| **[`@DeedleFake` 案](https://github.com/golang/go/issues/49189#issuecomment-953229306)** | キー型と値型を<br>分離  | `Key[myKey, int]{}`                                   | キー型の定義が冗長    |

[`@dsnet` 案](https://github.com/golang/go/issues/49189#issue-1037731122) には大きな欠点はなく，素晴らしいです。今後ビルトイン実装に入ってくるんだったらもうこれ一択でしょう。

# 私の実装

https://pkg.go.dev/github.com/mpyw/feature

私の実装はかなりそれに近いですが，機能的な差異としては以下のような観点があります：

## 名前なしキーにも対応

怠惰な人向けに，文字列で名前をつけなくていい [`New()`](https://pkg.go.dev/github.com/mpyw/feature#example-New) [`NewBool()`](https://pkg.go.dev/github.com/mpyw/feature#example-NewBool) も用意しています。 `var` 宣言の変数シンボルだけで済む！~~（別にこれ嬉しくなくね？）~~

```go
// 通常キー
var NamedFeature = feature.NewNamed[bool]("named-feature")

// 名前なしキー
var AnonFeature = feature.New[bool]()
```

もし名前なしの場合は，デバッグ時にコールサイト情報を **`anonymous(file:line)@addr`** 形式で付与します。

```go
var AnonFeature = feature.New[bool]()

fmt.Println(AnonFeature)
// Output: anonymous(/path/to/file.go:42)@0x14000010098
```

技術的には [`runtime.Caller`](https://pkg.go.dev/runtime#Caller) を使って定義位置を取得しています。これで「このフラグどこで定義されてるんだ？」がすぐわかります。

実装にあたり， [`New()`](https://pkg.go.dev/github.com/mpyw/feature#example-New) [`NewBool()`](https://pkg.go.dev/github.com/mpyw/feature#example-NewBool) [`NewNamed()`](https://pkg.go.dev/github.com/mpyw/feature#example-NewNamed) [`NewNamedBool()`](https://pkg.go.dev/github.com/mpyw/feature#example-NewNamedBool) とヘルパー含み 4 種類関数がある中で，何番目のトレースを取るかの調整が面倒だと感じました。これを簡単に解決するために，[ライブラリ内専用の unexported な調整関数](https://github.com/mpyw/feature/blob/13a0bcf31d1a893a11e45d34e6c1dd624687e43e/feature.go#L176-L180) を導入し， [生成用関数に含める](https://github.com/mpyw/feature/blob/13a0bcf31d1a893a11e45d34e6c1dd624687e43e/feature.go#L264) ことで，マジックナンバーを極力入れなくていいようにしています。

## 中間状態を構造体に

直接最終的な値を取得するのが基本的ですが， `.Inspect()` を使うと中間状態を取得でき，その [`feature.Inspection`](https://pkg.go.dev/github.com/mpyw/feature#Inspection) 型はそのままわかり易くログに表示可能です。この機能は Laravel の Authorization Gate の [Response](https://laravel.com/docs/12.x/authorization#gate-responses) から着想を得ています。

```go
var MaxItems = feature.NewNamed[int]("max-items")
inspection := MaxItems.Inspect(ctx)

fmt.Println(inspection)           // Output: "max-items: 100" or "max-items: <not set>"
fmt.Println(inspection.MustGet()) // Output: 100 or panic
fmt.Println(inspection.IsSet())   // Output: true or false
```

## ブール値の専用ラップ

ブール値は使用頻度が高いので， [`Key`](https://pkg.go.dev/github.com/mpyw/feature#Key) を embed したブール型専用の [`BoolKey`](https://pkg.go.dev/github.com/mpyw/feature#BoolKey) を用意しています。直感的な `.Enabled()` / `.Disabled()` メソッドで呼べるようにしました。

```go
var EnableSomething = feature.NewNamedBool("enable-something")

if EnableSomething.Enabled(ctx) {
    // 有効化されている場合の処理
}
if EnableSomething.Disabled(ctx) {
    // 無効化されている場合の処理
}
if EnableSomething.ExplicitlyDisabled(ctx) {
    // 明示的に無効化されている場合の処理
    // （未設定と区別できる）
}
```

## 「unexported なフィールドを持つ構造体」ではなく「Sealed Interface」を採用

[`@dsnet` 案](https://github.com/golang/go/issues/49189#issue-1037731122) では `Key` は構造体として定義し， `name` を unexported フィールドにすることで堅牢性を確保しています。

> ```go
> type Key[Value any] struct { name *string }
> ```

一方，私の実装では **Sealed Interface パターン** を採用しています:

https://taxio.hatenablog.com/entry/2020/12/08/020000

```go
type Key[V any] interface {
    WithValue(ctx context.Context, value V) context.Context
    Get(ctx context.Context) V
    TryGet(ctx context.Context) (V, bool)
    // ... 他のメソッド
    downcast() key[V]  // ← unexported メソッドがあるため，外部から実装できない
}

type key[V any] struct {  // ← unexported な実装により，外部から初期化できない
    name  string
    ident *opaque　// ← ここに new(opaque) で生成されるユニークポインタを持つ
}

// 名前としての string 引数は必須ではないので，汎用的に ident 専用の型を用意している
type opaque struct {
    _ byte  // ← 空構造体は同一アドレスに最適化されてしまうため，回避のために 1 バイトのデータを与える
}
```

ポイントは [`downcast()`](https://github.com/mpyw/feature/blob/13a0bcf31d1a893a11e45d34e6c1dd624687e43e/feature.go#L113-L115) という **unexported メソッド** を interface に含めていること。これにより，以下のような長所が生まれます。

- パッケージ外から [`Key[V]`](https://pkg.go.dev/github.com/mpyw/feature#Key) interface を実装することが **不可能** になる
- 利用者は必ず [`feature.New()`](https://pkg.go.dev/github.com/mpyw/feature#New) 等を経由してキーを生成する必要があり， **勝手に構造体を初期化されて不整合な状態になることを防げる**

:::message alert
先程 

> （後述）

としていた [`@dsnet` 案](https://github.com/golang/go/issues/49189#issue-1037731122) の唯一の懸念点がこちらになります。

「勝手に初期化される」について説明します。 Go はコンストラクタ関数を無視して型を初期化できてしまい，その際に適用されるルールはこうなります。

- ❌️ unexported フィールドがあるから初期化できない
- ✅️ **unexported フィールドは常にゼロ値で初期化される**

[`@dsnet` 案](https://github.com/golang/go/issues/49189#issue-1037731122) においては，コンストラクタ関数を経由しない誤った初期化を行うと

```go
badKeyX := &Key[int]{}     // name *string が nil
badKeyY := &Key[string]{}  // name *string が nil
```

このようにどちらも識別子が `(*string)(nil)` つまりゼロ値になってしまい，コンテキストキーとして衝突が発生してしまいます。

***「あれ？ `nil` をコンテキストキーに使おうとしたら panic になるんじゃないの？」***

と思うかもしれませんが， `(*T)(nil)` と `nil` では挙動が異なり，**前者の場合はゼロ値とみなして使えてしまい**，後者の場合は panic します。どちらにせよ問題ですが，不具合に気づきにくい前者のほうがある意味凶悪ですね。
:::

Sealed Interface では実装対象の構造体は unexported であるため，利用者による勝手な構造体初期化が完全に不可能であり，その interface の実装を利用者側が増やすこともできないので，利用法を制限したい場面の設計としては非常に堅牢です。但し， Go の文化として

***「受けるのは interface，返すのは struct」***

という慣習があるため，サードパーティライブラリとしては問題なくても，言語仕様としては採用されにくいかもしれません。

更にもう 1 つ長所を挙げるとすれば，構造体の場合は

```go
type Key[V any] struct {
    // ...
}

type BoolKey struct {
    Key[bool] // Embedded Type
    
    // ...
}
```

とすると `BoolKey` と `Key[bool]` は，メソッドの委譲とキャストが可能なだけで，型システム的には別物扱いになってしまうところを，

```go
type Key[V any] interface {
    // ...
}

type BoolKey interface {
    Key[bool]  // Embedded Interface
    
    // ...
}
```

とした場合は **[`BoolKey`](https://pkg.go.dev/github.com/mpyw/feature#BoolKey) は [`Key[bool]`](https://pkg.go.dev/github.com/mpyw/feature#Key) としても扱える** ため，相互運用性が高くなります。細かいですが，微かな長所として挙げてもいいと思います。

# 設計過程で発生した注意点

## 空構造体の罠

上述した [`opaque`](https://github.com/mpyw/feature/blob/13a0bcf31d1a893a11e45d34e6c1dd624687e43e/feature.go#L392-L396) を最初は空構造体にしていました。自信満々で書き上げた衝突回避確認のテストが無惨にも大失敗していて，何事かと思ったら…既にコメントでちらっと説明したように， [`opaque`](https://github.com/mpyw/feature/blob/13a0bcf31d1a893a11e45d34e6c1dd624687e43e/feature.go#L392-L396) 構造体へのポインタが全部同じアドレスになってしまいました。

```go
type opaque struct{}  // ❌ これだと全部同じアドレス

a := new(opaque)
b := new(opaque)

fmt.Printf("%p %p\n", a, b)  // 同じアドレスが出力される!
```

Go ではゼロサイズの構造体へのポインタは，コンパイラの最適化によって一箇所に集められちゃうんですね。 **これは構造体フィールドに `_ byte` を追加することで回避できます。** Claude Code に教えてもらったテクニックです，素晴らしい。~~ダーティハックにも程があるだろ~~

### noCopy ハックをやめた話

[`opaque`](https://github.com/mpyw/feature/blob/13a0bcf31d1a893a11e45d34e6c1dd624687e43e/feature.go#L392-L396) 導入よりもさらに前，最初は `key` 構造体自身をポインタとして引き回して，それをコンテキストキーにしていました。コピーされると困るので，以下で紹介されている **noCopy ハック** を入れていました。

https://devlights.hatenablog.com/entry/2024/11/19/073000

```go
// 初期の設計（ボツ）
type key[V any] struct {
    noCopy noCopy  // コピー警告対象にする
    // ...
}

func New[V any]() *key[V] {
    return &key[V]{
        // ...
    }
}

func (k *key[V]) WithValue(ctx context.Context, value V) context.Context {
    return context.WithValue(ctx, k, value)  // k 自身をキーに
}

// go vet 用の noCopy ハック
type noCopy struct{}
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

**コピーされると，アドレスが新しくアサインされることにより別のキーになってしまう致命的な欠点**があったため，このように対策していたのですが

***「内部にポインタを隠蔽して，外側はコピーされても大丈夫にしたほうが筋よくない？」***

と思い直して，今の設計になりました。シンプルになったし，利用者は何も考えなくていい。結果的に [`@dsnet` 案](https://github.com/golang/go/issues/49189#issue-1037731122) と同じ思想に落ち着いた形です。最終形が筋悪くなくて良かったぁ…

# 最後に

ちなみに業務ではこんな感じで投入しました:

*「検索をリッチにしたかったのでインデックスの効く全文検索の仕組みを導入したけど，影響範囲を小さくしたいからまず特定エンドポイントだけ有効にしたい！」*

```go
package features

import "github.com/mpyw/feature"

var FulltextIndexedSearch = feature.NewNamedBool("fulltext-indexed-search")
```

```go
// HTTP Handler 層（浅いところ）
func (h *Handler) Search(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx = features.FulltextIndexedSearch.WithEnabled(ctx)
    // ...
}
```

```go
// Infra 層（深いところ）
func (q *Querier) Search(ctx context.Context, query string) ([]Item, error) {
    if features.FulltextIndexedSearch.Enabled(ctx) {
        // Fulltext Indexed Search ロジック
    } else {
        // 従来の LIKE 検索の Search ロジック
    }
}
```

やっぱりこういうときコンテキスト超便利ですよね。是非活用しましょう。

小さいライブラリですが，地味ながら開発のお役に立てると思います。よかったら使ってみてください。

https://pkg.go.dev/github.com/mpyw/feature
