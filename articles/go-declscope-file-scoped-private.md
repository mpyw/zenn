---
title: "AI が書く Go コードの品質を劇的に向上させる Linter: “declscope”"
emoji: "✨"
type: "tech"
topics: ["go", "linter", "静的解析", "ai", "oss"]
published: true
publication_name: "yumemi_inc"
---

*旧題: 「Go のフラットなパッケージにファイル単位の private を持ち込む Linter “declscope”」*

# はじめに

## Go は「誰が書いても同じようになる」言語

Go には表現の選択肢が多くありません。三項演算子もなければ例外もなく，メタプログラミングで遊ぶ文化も薄い。 `if err != nil` を延々と書き，`for` を回し，構造体を素直に組み立てる。同じお題を 10 人に渡せば，([samber/lo](https://github.com/samber/lo) でも入れたりしなければ) 10 人ともよく似たコードを書いてくるでしょう。

この素朴さは長らく **記述量の多さ** という代償とセットで語られてきました。他の言語なら 1 行で済むことに 5 行かかる，と。手で書く人間にとって，これは明確な不利でしたね。

ところが，コードの大半を AI エージェントが書くようになった今，この代償はほとんど意味を失いました。定型を打ち込むコストが実質ゼロに近づいた一方で，**選択肢が少なく，読めば意味が一意に決まる** という性質のほうが重要です。生成されたコードが読んで分かる，差分が追える，エージェントが既存コードから書き方を推測しやすい — 素朴さは AI 時代に評価が反転した特性です。

## では， Go のコード品質は何で決まるのか

細部の書きぶりで差がつかないということは，裏を返すと **品質の勝負どころが細部にない** ということです。気の利いたショートハンドや言語機能の使いこなしは，そもそも Go には持ち込む余地が小さいですし，あったとしても **AI に任せてしまってよい部分** です。

残るのは大局のほう。すなわち **どの宣言をどこに置くか，何と何の間に境界を引くか** という，コードベース全体の整理の問題です。ここは AI が勝手に整えてくれる領域ではありません。むしろエージェントは目の前に見えているものを素直に使って書き進めるので，放っておけば整理は崩れていく一方です。

そして Go において，この「整理」のために与えられている道具は **パッケージだけ** です。ところがそのパッケージが，大きなコードベースを整理する道具としては少々非力なのです。

## フラットパッケージという思想

思想の派閥はありますが，その 1 つとして Go には「パッケージは少なく，大きく」という **フラットパッケージ** と呼ばれる作法があります。これに反し，境界を作りたいからといって安易にパッケージを割ると，それなりの代償を払うことになりますよね。

- **循環参照** エラーと，それを解消するためだけに生えてくる依存性逆転のためのインタフェース
- 分割した両者が共有する名前は，すべて Exported にしなければならない。2 つのファイルの間に境界が欲しかっただけなのに，API を公開することになる。露出を絞ろうとすると，**境界を引くたびに `internal/` 階層を挟む** 羽目になる
- 引いた境界が間違っていたときのコストが違う。宣言をファイル間で移動するのは影響が少ないが，パッケージ間で動かすのは破壊的変更になりやすい

多くの Go のコードはフラットなままのほうが健全と言われがちです。ところがフラットにすれば問題が消えるわけではなく，フラットパッケージ思想も諸刃の剣です。Go の可視性には以下の 2 段階しかありません。

| レベル | 到達範囲 |
|:---|:---|
| Exported | **import した全パッケージから** |
| unexported | **パッケージ内の全ファイル** |
| ファイル単位 | **存在しない** |

**unexported は「このファイルだけ」ではありません。** フラットなパッケージでは，どんなに小さなヘルパーもパッケージ全体から見える名前で，どんなフィールドもパッケージ全体から書き換えられます。

> *このヘルパーはこのファイルのものだから，他のファイルから呼ばないでね*

こういった規約自体は有効かもしれませんが，それが存在している場所はパッケージを書いた人間の頭の中だけです。コメントで都度記すこともできますが，結局のところコンパイラはそれを知りません。

## AI エージェントは規律を守れるのか？

そして Agentic AI 全盛期の今，この暗黙の了解から綻びが生まれやすくなりました。エージェントはスコープに見えている unexported なヘルパーを見つけたら，素直に呼びます。unexported なフィールドが見えていたら，素直に書き込みます。そのどれもがコンパイルを通り，レビューが雑であれば通過してしまいます。 AI は書くスピードが非常に速いので，負債が積み上がるスピードも段違いです。

ここで大事なのは，**エージェントは規約を破っているというより，見えている API を使っているだけ** だということです。エージェントに届く 2 つの入力は，届き方がまるで違います。

| エージェントへの入力 | 届き方 |
|:---|:---|
| コメントや `CLAUDE.md` に書いた規約 | 読まれるかどうかは **確率的** |
| パッケージに見えている宣言 | **必ず見えるし，必ずコンパイルが通る** |

:::message
「CLAUDE.md に書けばいいじゃん」と思いましたか？書きました。書いた上で，完全性は保てませんでした。自然言語の規約は，守られたかどうかを決定論的に判定できない以上，どうしても確率的にしか効きません。
:::

だったら，規約のほうを機械的に検査させればいいですよね。パッケージはフラットなまま，境界はその内側に置いたまま，専用の Linter でチェックする。エージェントが境界を越えたときには「何を越えたのか」と「どう直すのか」が突き付けられ，次のエージェントがそれを読む。このサイクルを回せばいいのです。

というわけで，作りました。

https://github.com/mpyw/declscope

```bash
mise use -g "github:mpyw/declscope"   # go install や go tool でも入ります
declscope ./...
```

# 何が報告されるのか

`database` という 1 つのパッケージに，リポジトリの実装を並べる構成を考えます。よくありますよね。

```go:user_repository.go
package database

type UserRepository struct{ db *sql.DB }

func (r *UserRepository) Find(ctx context.Context, id int64) (*domain.User, error) {
    row := r.db.QueryRowContext(ctx, `SELECT id, email FROM users WHERE id = ?`, id)
    return scanUser(row)
}

// 生の結果セットをドメインエンティティに変換する
func scanUser(row *sql.Row) (*domain.User, error) {
    var u domain.User
    var email string
    if err := row.Scan(&u.ID, &email); err != nil {
        return nil, err
    }
    u.Email = normalizeEmail(email)
    return &u, nil
}

func normalizeEmail(s string) string {
    return strings.ToLower(strings.TrimSpace(s))
}
```

```go:order_repository.go
package database

type OrderRepository struct{ db *sql.DB }

func (r *OrderRepository) List(ctx context.Context, userID int64) ([]domain.Order, error) {
    // 略：クエリを投げて rows を得る
    for rows.Next() {
        o, err := scanOrder(rows)
        // 略
    }
    return orders, rows.Err()
}

// 生の結果セットをドメインエンティティに変換する
func scanOrder(rows *sql.Rows) (domain.Order, error) {
    var o domain.Order
    var email string
    if err := rows.Scan(&o.ID, &email); err != nil {
        return domain.Order{}, err
    }
    o.BuyerEmail = normalizeEmail(email)
    return o, nil
}
```

この 2 つのヘルパーは，やっていることが同じです。生の結果セットをドメインエンティティに詰め替えるだけ。だから本当はどちらも `scan` と名付けたい。

でもフラットなパッケージでは **`scan` は 1 つしか宣言できません。** だから都度 `scanUser` / `scanOrder` と書き分けています。

ここで起きていることをよく見てください。名前に付けた `User` / `Order` は，「これはどちらのリポジトリのものか」という所有の表明です。衝突を避けるために仕方なくやったことが，結果的に規約になっているわけですね。

そして **その規約をコンパイラは知りません。**

## AI エージェントが起こす越境

`normalizeEmail` は `user_repository.go` に書かれた，`scanUser` のためのヘルパーでした。ところが `scanOrder` がそれを呼んでいます。

```go
o.BuyerEmail = normalizeEmail(email) // ← user_repository.go のもの
```

コンパイラはこれを受け入れます。同じパッケージの unexported な関数を呼んでいるだけですから。レビューでも気付きにくいですし，そもそも **AI エージェントはスコープに見えているものを素直に使います。**

**declscope はこれを検知します。**

```console
$ declscope ./...
user_repository.go:21:6: func normalizeEmail is private to namespace "userRepository", but is used from namespace "orderRepository"
order_repository.go:21:20:      used here, in namespace "orderRepository"
```

## 越境の解消には答えが 2 つある

| 答え | やり方 |
|:---|:---|
| 境界を守る | 呼び出しを namespace の内側に移す |
| 意図的に共有する | `//declscope:package` を書く。 `-fix` が挿入してくれる |

今回はどちらでしょうか。メールアドレスの正規化は，どちらのリポジトリのものでもありません。となると本当にやりたいのは **第三の場所への切り出し** — `email.go` を作ってそこに移すこと，のはずです。

ただし，`boundary` は切り出しただけでは黙りません。`email.go` に置いたところで，`user` と `order` の両方から使われることに変わりはないからです。切り出したうえで，**共有物だと宣言する** 必要があります。

```go:email.go
package database

//declscope:package
func normalizeEmail(s string) string {
    return strings.ToLower(strings.TrimSpace(s))
}
```

この 1 行が **「これは共有物である」という宣言** としてソースに残ります。次にこのファイルを開いた人間もエージェントも，それを最初に読みます。

:::message
`//declscope:package` は `-fix` でも挿入できますが，**今回のケースでは使いません。** `-fix` ができるのは宣言をその場で広げることだけで，ファイルを跨いで動かすことはできないからです。任せると `normalizeEmail` は `user_repository.go` に残ったまま共有物になります。

`boundary` の `-fix` は常に広げる方向にしか働きません。ツールが機械的に適用できる修正がそれしかないからです。**どこに置くべきかという判断こそ，AI エージェントにやらせたいところ** です。
:::

# 知っておく必要があるのは 2 つだけ

## ファイル名が namespace

**namespace** は， `private` な宣言を使ってよい範囲の単位です。既定では 1 ファイルが 1 つの namespace で，ファイル名を lowerCamelCase 化した名前が付きます（`user_repository.go` → `userRepository`）。

:::message
`*_test.go` や `*_windows.go` `*_darwin.go` などの Go の慣習に従うサフィックスは除外されます。
:::

1 つの単位が複数ファイルにまたがる場合は，package 節の前にディレクティブを書いて **共有 namespace** に参加させます。

```go:user_repository.go
//declscope:namespace user

package database
```

### core namespace

もう 1 つ，**名前を持たない namespace** を 1 パッケージにつき 1 つだけ作れます。

```go:client.go
//declscope:core

package transport

func doSomething() int { return 1 }
```

複数のファイルが `//declscope:core` を書いてよく，それらは 1 つの core namespace を共有します。

**`boundary` から見れば，core は単なる 1 つの無名の namespace でしかありません。** 特別扱いは何もなく，core の宣言も既定では core の中だけで private です。core の外から触れば普通に報告されます。

```console
client.go:4:6: func doSomething is private to the core namespace, but is used from namespace "other"
```

意味を持つのは後述の `qualify` のほうです。**core には名前がないので，`qualify` は core の宣言に何も要求しません。** パッケージの主題そのものを書くファイルや，どこからでも使われるユーティリティに，意味のないプレフィックスを強いないための逃げ道です。

:::message
 core に入れたファイル同士は 1 つの namespace を共有するので，互いの private な宣言に触れるようになります。何でも core に放り込むと，そこだけフラットなパッケージに戻るので，濫用にご注意ください。
:::

## scope は `package` と `private` の 2 つだけ

| scope | 意味 | Rust での相当物 |
|:---|:---|:---|
| `package` | パッケージ内のどこからでも使える | `pub(super)` |
| `private` | 自分の namespace の中でだけ使える | 修飾子なし |

`public` はありません。Go は既にそれを大文字で綴っていますし，パッケージの外側での使用は declscope の調査対象外です。

既定では Exported なものが `package`，それ以外が `private` です。

# ルールは 5 つ

| ルール | 何を問うか | 既定 |
|:---|:---|:---|
| **`boundary`** | この namespace から，あの namespace のものに触ってよいか | 既定で ON |
| **`qualify`** | シンボル名のどこかに namespace が含まれているか | 既定で OFF |
| `surplus` | その共有宣言，本当に必要か | 既定で `loose` |
| `unused` | そのディレクティブ，何か決めているか | 既定で `loose` |
| `directive` | そのディレクティブ，正しく書けているか | 常に ON |

下の 3 つは説明不要でしょう。`surplus` は他の namespace からの使用が 1 つも見えない `//declscope:package` を，`unused` は消してもスコープが何も変わらないディレクティブや，何も黙らせていない `//declscope:ignore` を報告します。どれも「書いたものが無駄になっていないか」の監査です。`directive` は書式の崩れたディレクティブや，未知・矛盾したディレクティブを報告します。ディレクティブとして読まれるのは `//declscope:name` の形だけで，`// declscope:package` のようにスペースを入れたものは効かずに報告されます。`surplus` と `unused` には，より厳しく判定する `strict` モードもあります（後述）。

## `boundary` ルール

冒頭の例で見たものです。**`private` な宣言が，自分の namespace の外から使われている** ことを報告します。

対象はパッケージレベルの宣言と，型のメンバ（struct のフィールド，interface のメソッド名）です。そして namespace の決まり方に例外はありません。**その宣言が物理的に書かれているファイル**，それだけです。

- 型のメンバは，その型の宣言の内側に書かれているので，**型があるファイル** のもの
- レシーバを持つメソッドは，型がどこにあろうと **それを書いたファイル** のもの

型のメソッドを複数ファイルに分けて書くのは，Go では普通のことです。同じ `database` パッケージに，簡単なクエリビルダを足してみましょう。

```go:statement.go
package database

// Statement はクエリの組み立て途中の状態を持つ
type Statement struct {
    Table  string
    wheres []string
    args   []any
}

func (s *Statement) Build() (string, []any) {
    q := "SELECT * FROM " + s.Table
    if len(s.wheres) > 0 {
        q += " WHERE " + strings.Join(s.wheres, " AND ")
    }
    return q, s.args
}
```

```go:query.go
package database

// 条件を積む DSL。Statement のメソッドだが，まとめてここに置いている
func (s *Statement) Where(cond string, args ...any) *Statement {
    s.wheres = append(s.wheres, cond)
    s.args = append(s.args, args...)
    return s
}
```

型の宣言を `statement.go` に，条件を積む DSL を `query.go` に置く。有名 OSS にも見られる構成です。

```console
$ declscope ./...
statement.go:6:5: field Statement.wheres is private to namespace "statement", but is used from namespace "query"
query.go:5:5:   used here, in namespace "query"
query.go:5:23:  used here, in namespace "query"
statement.go:7:5: field Statement.args is private to namespace "statement", but is used from namespace "query"
query.go:6:5:   used here, in namespace "query"
query.go:6:21:  used here, in namespace "query"
```

`Where` を `query.go` に置いたこと自体は咎められていません。報告されたのは `wheres` と `args` のほうで，しかも **宣言元である `statement.go` に対して** 記録されています。

この 2 つのファイルは実質的に 1 つの関心範囲を 2 つに割ったものなので，名前空間を 1 つに束ねると良いでしょう。

```go:query.go
//declscope:namespace statement

package database
```

これで `query.go` は `statement` namespace に参加し，エラーは消えます。

## `surplus` ルールの `strict` モード

既定は `loose` ですが，**私は `strict` にすることをおすすめします。** `surplus` は `loose` では **ディレクティブ単位** で判定します。1 つの `//declscope:package` が束ねる宣言のうち，1 つでも他の namespace から使われていれば，そのディレクティブ全体が黙ります。

これで困るのが **複数の namespace から使われる構造体** です。型に `//declscope:package` を付けるとフィールドもまとめて `package` になりますが，他から読まれるのはその一部だけ，ということはよくあります。

```go:account.go
package bank

//declscope:package
type account struct {
    id      int
    balance int
}

func accountDeposit(a *account, n int) { a.balance += n }
```

```go:ledger.go
package bank

// ledger から読むのは id だけ
func ledgerKey(a account) int { return a.id }
```

`loose` ではこれは何も報告されません。`id` が使われているからです。しかし `balance` は必要以上に広く開いたままです。

`rules.surplus: strict` にすると **宣言 1 つ 1 つ** を判定し，これを報告します。

```console
$ declscope ./...
account.go:7:2: field account.balance takes package scope from //declscope:package on account, but no use from another namespace is visible to declscope
```

こちらは `-fix` で **狭める方向の修正** が入ります。

```go:account.go
//declscope:package
type account struct {
    id int
    //declscope:private
    balance int
}
```

型は共有物として宣言し，他の namespace が読まないフィールドだけを `private` に戻す。**共有する範囲を必要最小限に保つ** ためのパターンです。

:::message
慣習としては private なフィールドを後ろにまとめるのが読みやすいですが，`-fix` はフィールドを並べ替えません。キーなしの複合リテラルやバイナリエンコーディング， `unsafe` のオフセットなど，フィールドの順序が観測される場面があるからです。問題がない場合は自分で並べ替えてください。
:::

型のフィールドのほか，`var` / `const` / `type` ブロック内の各 spec や，ファイル全体に掛けた `//declscope:package` の下の各宣言も同じように判定されます。

## `unused` ルールの `strict` モード

既定は `loose` ですが，**私は `strict` にすることをおすすめします。** `unused` は `loose` では，**どの設定でも** 消してスコープが変わらないディレクティブだけを報告します。`defaults.unexported` の値を変えれば効くようになるディレクティブは，黙ります。

これで残るのが **既定値を書き写しただけのディレクティブ** です。unexported な構造体のフィールドは既定で `private` なので，次の `//declscope:private` は今の設定では何も変えていません。

```go:user.go
package app

type user struct {
	//declscope:private
	name string
}

func userName(u user) string { return u.name }
```

`rules.unused: strict` にすると **今の設定で** 判定し，これを報告します。

```console
$ declscope ./...
user.go:4:2: unused //declscope:private on user.name: it already has private scope
```

`strict` の報告には `-fix` が付き，ディレクティブの行を消します。

```diff
 type user struct {
-	//declscope:private
 	name string
 }
```

消すと他の報告が変わってしまう場合，たとえば別の namespace がその宣言を使っていて `boundary` の報告がそのディレクティブを名指ししている場合は，報告だけ残して `-fix` は付きません。

:::message alert
`strict` では `defaults.unexported` を変えると報告が変わります。新しい既定値と同じことを書いたディレクティブがすべて報告されます。`-fix` を 1 回走らせれば消えます。
:::

未使用の報告を黙らせたいときは `//declscope:ignore unused` と書きます。`//declscope:ignore directive` では黙らず，その ignore 自体が「何も黙らせていない」と報告されます。

## `qualify` ルール

既定で OFF ですが，**私は ON にすることをおすすめします。** `boundary` が 「この namespace から，あの namespace のものに触ってよいか」 を問うのに対し，`qualify` は **「シンボル名のどこかに namespace が含まれているか」** を問います。パッケージレベルの宣言に要求するのは，これだけです。

:::message
**プレフィックスの強制ではありません。** 診断が提案するのは既定でプレフィックスですが，それは条件を満たす答えの 1 つでしかなく，従う義務はありません。「どこかに含まれていればよい」ので，自然な語順のまま満たせることのほうが多いはずです。
:::

:::message
この命名ルールは `boundary` ルールに対しては何の影響も与えません。
:::

また **構造体フィールドと，同一 namespace 内のメソッドは対象外** です。

さきほどのクエリビルダで，両者が並んでいる様子を見てみましょう。`Build` は型と同じ `statement.go` に，`Where` は `query.go` にあります。

```go:statement.go
type Statement struct {
    Table  string
    wheres []string // 対象外
}

func (s *Statement) Build() (string, []any) { /* ... */ }   // 対象外
```

```go:query.go
func (s *Statement) Where(cond string) *Statement { /* ... */ }   // 対象
```

```console
$ declscope ./...
query.go:4:21: method Where does not carry namespace "query" anywhere in its name;
               rename it to QueryWhere, or to another name that carries "query"
```

`Build` には何も要求されず，`Where` だけが報告されました。**同じ型の，同じ形をしたメソッドなのに扱いが違います。**

分かれ目は **外部メソッド** かどうかです。

- 構造体と同じ namespace にあるメソッドは，構造体の命名規則だけで十分な識別情報を提供しているため，二重にメソッド名にも命名規則を課されることはありません。
- 一方で，構造体とは異なる namespace にメソッドがある場合は **外部メソッド** として扱われ，その namespace の名前を含むことが要求されます。

尤も，自動提案された `QueryWhere` が答えでないのは明らかです。以下のどちらかがよいのではないでしょうか。

- 名前空間を `statement` 1 つに束ねてしまう
- `query.go` を更に細かいトピックである `where.go` にリネームする

---

もう一つ例を見るため，冒頭の `user_repository.go` の話に戻りましょう。

> ```go:user_repository.go
> package database
> 
> type UserRepository struct{ db *sql.DB }
> 
> func (r *UserRepository) Find(ctx context.Context, id int64) (*domain.User, error) {
>     row := r.db.QueryRowContext(ctx, `SELECT id, email FROM users WHERE id = ?`, id)
>     return scanUser(row)
> }
> 
> // 生の結果セットをドメインエンティティに変換する
> func scanUser(row *sql.Row) (*domain.User, error) {
>     var u domain.User
>     var email string
>     if err := row.Scan(&u.ID, &email); err != nil {
>         return nil, err
>     }
>     u.Email = normalizeEmail(email)
>     return &u, nil
> }
> 
> func normalizeEmail(s string) string {
>     return strings.ToLower(strings.TrimSpace(s))
> }
> ```

このルールを ON にすると，まずこう言われます。関数 `scanUser` が改名対象として引っかかります。 

```console
$ declscope ./...
user_repository.go:11:6: func scanUser does not carry namespace "userRepository" anywhere in its name;
                         rename it to userRepositoryScanUser, or to another name that carries "userRepository"
```

`userRepositoryScanUser` はさすがにひどいですよね。でも **この指摘自体は正しくて，おかしいのは提案されたリネームのほう** です。

よく考えてください。このファイルが担当している関心範囲は `userRepository` でしょうか？違いますよね，`user` です。 `Repository` はファイル名に入っているだけで，関心範囲を表す語ではありません。

```go:user_repository.go
//declscope:namespace user

package database
```

これで namespace が `user` になり，`scanUser` は `user` を含むので黙ります。`order_repository.go` も同様です。

残るのは `normalizeEmail` です。これは `user` を名乗れません。そして **名乗れないのは，それが user のものではないから** です。

```go:email.go
package database

//declscope:package
func normalizeEmail(s string) string {
    return strings.ToLower(strings.TrimSpace(s))
}
```

`email.go` に移せば namespace は `email` になり，`normalizeEmail` はそれを含むのでエラーはなくなります。

これでこのパッケージのエラーはすべて解消されました。**`qualify` が問うているのは「名前を変えろ」ではなく，「その名前と，それが置かれた場所は噛み合っているか」** です。答えは名前を変えることかもしれませんし，namespace を変えることかもしれませんし，ファイルを分けることかもしれません。

### 推奨設定

declscope が自分自身に課しているのと同じ設定です。

```yaml:.declscope.yaml
rules:
  naming:
    qualify: ondemand   # 2 つ目の namespace ができた時点で要求する
    exported: true      # Exported な宣言も対象にする
  surplus: strict       # 共有したが他から使われていない宣言を 1 つずつ報告する
  unused: strict        # 今の設定で何も変えていないディレクティブを報告して消す
```

`qualify: ondemand` は，namespace が 2 つ以上のパッケージで有効になり，1 つでは無効になります。全部に同じプレフィックスが付いたところで何も区別しないから無意味だ，という考えです。通常はこの設定がよいのではないでしょうか。

`exported: true` は公開 API にも関わるため，必ずしも導入が成功するとは限りませんが， internal 構成がメインのリポジトリであればそれほど大きな影響なく導入できるかもしれません。新規プロジェクトであればぜひ導入したいところです。

:::message
なお **Exported な宣言にリネームは提案されません。** 報告されるだけです。パッケージの外からの使用は declscope に見えないので，書き換えていいか判断できないからです。`-fix` を走らせても公開 API が勝手に変わることはありません。
:::

# 既存のコードベースに入れる

既に大量の越境があるコードベースに対しては， **baseline** を使います。

```console
$ declscope baseline ./...       # .declscope-baseline.yaml を書き出す
```

今ある違反が記録され，以後は **新しく増えたものだけ** が報告されます。エントリのキーはパッケージ・ルール・namespace・宣言名であって位置ではないので，ファイル内でコードが動いても生き残ります。

そして baseline は **抑制はしますが，是認はしません。** ソースには何も書き込まれないので新しい宣言にはルールがそのまま適用されるし，エントリが消えるのは違反が直ったときだけです。片付いた分は `git diff` にそのまま出ます。

## 実際にやってみた

有名なフラットパッケージである [spf13/cobra](https://github.com/spf13/cobra) に declscope をかけて，**指摘がゼロになるまで直しきった** ものを fork 上の PR として置いてあります。コミットを追うと件数が段階的に落ちていくので，何にいくら効くのかが読めると思います。

https://github.com/mpyw-forks/spf13-cobra/pull/1

| 項目 | spf13/cobra |
|:---|---:|
| 対象 | 14 ファイル / 6,138 行 |
| 開始時の `boundary` / `qualify` | 35 / 119 |
| 単位を宣言した後 | 21 / 76 |
| 最終 | **0 / 0** |
| 差分 | 25 ファイル +291/-205 |

:::message alert
これは **本家に向けた提案ではありません。** base も head も自分の fork で，クローズ済みです。バグ報告ではなく，「ファイル分割と設計の対応関係を数えるとどう見えるか」の実例です。
:::

### 最も古いファイルだけが，リネームを忘れていた

読みどころを 1 つだけ挙げておきます。**cobra には，自分の規約から取り残されたファイルが 1 つありました。**

補完生成器は 5 つあります。zsh も fish も powershell も，そして bash の V2 でさえ，関数名で自分のシェル名を名乗っています。

```go
genZshComp
genFishComp
genPowerShellComp
genBashComp // V2
```

ところが **最も古い `bash_completions.go` だけが名乗っておらず**，一番汎用的な名前を占有していました。リネームで namespace を明示すると，こうなります。

```diff
- gen
+ genBashCommands
- writeFlag
+ writeBashFlag
- writeCommands
+ writeBashCommands
```

`writeFlag` が bash のものだと誰も書いていない以上，2 つ目の生成器は自分の `writeFlag` を持てません。このように declscope は，実在 OSS でも **フラットパッケージの地雷を検出できました。**

# AI 時代に必要なのは，判断を記録すること

冒頭の動機に戻りましょう。declscope は，**エージェントの編集が検査される場所ならどこでも** 走ります。CI でも，エージェントのビルドループの中でも同じです。

既存のコードベースなら最初に一度 baseline を取っておきます。するとエージェントには自分の編集が越えた境界だけが見えるようになります。

書かれただけの規約との違いは，2 点に集約されます。

| 性質 | エージェントへの効果 |
|:---|:---|
| 診断が越えた namespace を名指しする | なぜその使用が間違いなのかが伝わり，修復が機械的になる |
| ディレクティブが意図の永続的な記録になる | 次のエージェントは，判断を導出し直さずに継承する |

2 つ目が個人的には本命です。「このヘルパーは共有していい」という判断は，これまでレビューのコメント欄か，人間の記憶の中で消えていきました。declscope ではそれが `//declscope:package` という 1 行でソースに残り，**次にそのファイルを開いたエージェントが最初に読むもの** になります。

## 導入作業の知識も skill として同梱している

README は「declscope が何であるか」を書く場所です。しかし **「既存のリポジトリに入れるとき，エージェントが何を踏むか」はまったく別の知識** でした。後者を skill にして同梱しています。

```bash
gh skill install mpyw/declscope
```

中身は，例えばこういうことが書いてあります。

- **ビルドが失敗していると診断は 0 件になる。** そしてそれは成功と見分けがつかない。カウントを読む前に `go build ./...` を通せ
- **baseline があると全部 0 件になる。** 測定するときは先に退避しろ。でないとコードベースが綺麗に見える
- **`qualify` の 0 件は「綺麗」とは限らない。** 既定で OFF なので，設定を確認せずに 0 を見て「問題なし」と報告してはいけない。設定ファイルは解析対象パッケージから上へ探索されるので，リポジトリに複数あり得る。全部探せ
- **`//declscope:namespace` は package 節の前に書く。** 後ろに置くと黙って無効になる。何をしても診断が動かないときは，まず配置を疑え
- **設定はリポジトリオーナーの決定であって，エージェントが決めることではない。** 設定ファイルを書く前に聞いて，答えを待て

作業の順序も書いてあります。**`boundary` を先に，それも広げるのではなく境界を動かして片付けろ。そして測り直せ。** `boundary` を潰す過程で 2 つの namespace が 1 つに統合されると `ondemand` の条件から外れるので，`qualify` の指摘が連鎖的に消えることがあるからです。逆順にやると，消えるはずの指摘をリネームして回る羽目になります。`boundary` が片付いたら，`surplus: strict` を提案するところまで書いてあります。

# 限界

declscope は **1 度に 1 パッケージだけを読み，名前が綴られている場所だけを使用として数えます。** したがって以下は見えません。

- struct の値全体に対する操作（コピー・比較・ゼロ化はフィールドを 1 つも名指ししない）
- リフレクション， `//go:linkname` ，生成コード
- **誰も使っていない宣言**。 `boundary` は使用箇所を探すルールなので，未使用コードは何も報告しない

最後の 1 つについては [`deadcode`](https://pkg.go.dev/golang.org/x/tools/cmd/deadcode) などの未使用コード用 Linter と併用してください。パッケージ間の境界を引きたい場合は [`depguard`](https://github.com/OpenPeeDeeP/depguard) です。 `depguard` がパッケージグラフを正直に保ち，declscope が各パッケージの内側を正直に保ち，`deadcode` がどちらも必要としなくなったものを剥がす — という住み分けになります。

![Go のプログラムを入れ子のフレームで描いた図。api パッケージと database パッケージの間では depguard が「このパッケージはあのパッケージを import してよいか」を問い，api から database への緑の矢印と，database から api への赤い矢印に打ち消し線が引かれている。database の内側，user_repository.go と order_repository.go の間では declscope が「このファイルはあの宣言に手を伸ばしてよいか」を問い，scanOrder から normalizeEmail への赤い矢印に打ち消し線が引かれている。プログラムの縁では deadcode が「そもそも到達可能か」を問い，矢印がどこからも入っていない mail パッケージが unreachable と注記されて灰色になっている。](https://raw.githubusercontent.com/mpyw/declscope/main/docs/boundaries.png)

# まとめ

- **Go の可視性は 2 段階しかなく，ファイル単位の private はない。** フラットなパッケージを保つ限り，「このファイルのもの」という規約は人間の頭の中にしか存在しない
- **AI エージェントはその規約を知らないし，人間が書いていた頃より桁違いの速度で違反を犯してくる。** 自然言語の規約は，決定論的に判定できない以上どうしても確率的にしか効かない
- **declscope は，パッケージをフラットに保ったまま，その規約を決定論的に検査する。** 越境には「境界を守る」か「意図的な共有だと宣言する」かの 2 つの答えが必ずある
- **`qualify` も ON にするのがおすすめ。** ただしその指摘は「名前を変えろ」ではなく「そのファイルは，それを宣言する場所として妥当か？」を問うている
- **`surplus` と `unused` も `strict` がおすすめ。** 他から使われていない共有と，何も決めていないディレクティブを 1 つずつ洗い出し，`-fix` で片付けられる
- **AI 時代に必要なのは，判断を記録すること。** ディレクティブは人間が下した判断の永続的な記録になり，次のエージェントはそれを導出し直さずに継承する
- **導入手順そのものも skill として同梱した。** エージェントが読む前提の手順書を作者が書いて配れる時代になった

https://github.com/mpyw/declscope

使ってみて「ここが分かりにくい」「このルールは厳しすぎる」などありましたら，是非 Issue やコメント欄で教えてください。
