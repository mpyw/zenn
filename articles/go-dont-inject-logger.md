---
title: "なぜ Go ではロガーをコンストラクタ DI してはならないのか"
emoji: "🥹"
type: "tech"
topics: ["go", "php", "di", "logger", "アンチパターン"]
published: false
---

# はじめに

恐らく Go に関する記事を書くのはこれが初めてです。まだ業務経験としては浅いので，お手柔らかにお願いします。

さて，直近取り組んでいるプロジェクトの中で，大きな設計ミスをしてしまったという後悔をしていることがあります。それはロガーの取り回しについてです。 PHP と Laravel を主戦場としてきた私が，そこで得てきた知識を Go に応用しようとしたら，設計ミスで大火傷したエピソードについて語ります。みなさんは決して同じ轍を踏まないでください。

**対象読者: PHP（とくに Laravel や Symfony）でそこそこイケてる設計を実践してきた Go 初心者の方々**

# 問題のある実装パターン

## 共通実装

以下のような `applog` パッケージ上のロガー実装を考えましょう。ここでは Go 標準の `log.Logger` をラップしていますが，様々な実装に拡張できることを想定しています。

```go
package applog

import (
	"fmt"
	"log"
	"os"
)

type Logger interface {
	Info(message string)
	Error(message string)
}

func NewLogger() Logger {
	return &logger{
		inner: log.New(os.Stdout, "", log.LstdFlags),
	}
}

var _ Logger = (*logger)(nil)

type logger struct {
	inner *log.Logger
}

func (l *logger) Info(message string) {
	l.inner.Println(fmt.Sprintf("[INFO] %s", message))
}

func (l *logger) Error(message string) {
	l.inner.Println(fmt.Sprintf("[ERROR] %s", message))
}
```

## コンストラクタ DI するとどうなるか？

この `Logger` を様々なサービスに引き回すことを考えます。

全ての Go のプロジェクトで採用されるかどうかは分かりませんが，クリーンアーキテクチャなどでお馴染みの `Service` や `Repository` といった概念をそのまま踏襲した構造体・インタフェースを作ってみます。

```go
package users

import (
	"fmt"

	"example.com/myapp/applog"
)

// エンティティの定義

type User struct {
	ID   int
	Name string
}

// リポジトリの定義

type Repository interface {
	FindByID(id int) *User
}

func NewStubRepository(l applog.Logger) Repository {
	return &stubRepository{
		logger: l,
		users: map[int]*User{
			1: {ID: 1, Name: "Alice"},
			2: {ID: 2, Name: "Bob"},
		},
	}
}

var _ Repository = (*stubRepository)(nil)

type stubRepository struct {
	logger applog.Logger
	users  map[int]*User
}

func (r *stubRepository) FindByID(id int) *User {
	user, exists := r.users[id]
	if !exists {
		r.logger.Error("User not found")
		return nil
	}
	r.logger.Info(fmt.Sprintf("Fetched user with ID %d", id))
	return user
}

// サービスの定義

func NewService(logger applog.Logger, repo Repository) *Service {
	return &Service{logger: logger, repo: repo}
}

type Service struct {
	logger applog.Logger
	repo   Repository
}

func (s *Service) GetUserByID(id int) *User {
	s.logger.Info(fmt.Sprintf("Invoking repo.FindByID with %d", id))
	return s.repo.FindByID(id)
}
```

必要に応じて実体を差し替えられるように， `Repository` はインタフェースにしておきました。これはよくあるアプローチだと思います。 また Go にはクラスコンストラクタという概念がないので， `NewStubRepository` `NewService` 関数をそれと同等のものとして扱っています。 

そしてこれらを統合し，実際にアプリケーションのエントリポイントとなる `main` 関数を作ります。

```go
package main

import (
	"fmt"

	"example.com/myapp/applog"
	"example.com/myapp/users"
)

func main() {
	// サービスの組み立て
	// （実際のこのあたりのコードは DI ライブラリに任せることが多い）
	logger := applog.NewLogger()
	svc := users.NewService(logger, users.NewStubRepository(logger))
	
	// ユーザの取得
	fmt.Println(svc.GetUserByID(1))
	fmt.Println(svc.GetUserByID(2))
	fmt.Println(svc.GetUserByID(3))
}
```

さて，一通り実装は終わったように見えますが…いったい何が問題になるのでしょうか？

### 問題点: ロガーの引き回しを忘れると詰む

必然ですが，*「ここでログを取るなんて想定していなかった…」* みたいなことが起こると即死です。鬼のようなリファクタリングコストがあなたに降りかかります。

### 問題点: Go では PHP のように何でもかんでもクラス・インスタンスメソッドにしない

PHP では

*「メソッドじゃない関数を使うな！」*
*「静的メソッドはできるだけ使うな！インスタンスメソッドにしろ！」*

という文化が定着していると思いますが，そんな話は Go には通用しません。普通の関数にメソッドからロジックが委譲されることなんて腐る程あります。

それに， Go では **メソッド上で引数に対する型パラメータは宣言できない** という制約があります。

:::message
構造体上に型パラメータを定義し，それをレシーバ引数として参照することはできます。
:::

:::details ジェネリクスの利用例

```go
package main

import "fmt"

// ジェネリクスを使った普通の関数
func Swap[T any](a, b T) (T, T) {
	return b, a
}

// ジェネリクスを使って定義された構造体
type Box[T any] struct {
	Value T
}

// レシーバ引数としてジェネリクスを使用する構造体のメソッド
func (b *Box[T]) Get() T {
	return b.Value
}

// メソッド上で引数に対する型パラメータは宣言できない
/*
func (b Box) SetNewValue[T any](v T) {
	// このようにメソッド引数に対して直接型パラメータを宣言することはできません。
}
*/

func main() {
	a, b := 5, 10
	fmt.Println("Before swap:", a, b)
	a, b = Swap(a, b)
	fmt.Println("After swap:", a, b)

	intBox := Box[int]{Value: 42}
	fmt.Println("Box value:", intBox.Get())
}
```
:::

このような事情を踏まえると，メソッドではなく関数を採用する場面もたくさんあり，関数を採用したときに **毎回ロガーを引数でリレーすることになります**。端的に言って地獄です。

### 問題点: DI ライブラリが事実上必須になってしまう

Go は比較的 YAGNI 思想が強く顕れている言語です。規模によっては DI ライブラリを採用せず，自前で DI することも多いでしょう。 しかし， Logger ごときをコンストラクタインジェクションしていたら，さぞかし骨が折れることでしょう。

> ```go
> logger := applog.NewLogger()
> svc := users.NewService(logger, users.NewStubRepository(logger))
> ```

手作業で書くならこんなコード見たくないですよね？こうあってほしいですよね？

```go
svc := users.NewService(users.NewStubRepository())
```

それに，ロガーは使用されるパッケージがかなり広範囲に及ぶと思います。考えるだけで頭が痛くなりますね。

## グローバル変数を使うとどうなるか？

じゃあ DI なんて考えずにグローバル変数を使っちゃえばいいじゃん！ということでその実装がこちら。

```go
package applog

import (
	// ...
)

var DefaultLogger = NewLogger()

func Info(message string) {
	DefaultLogger.Info(message)
}

func Error(message string) {
	DefaultLogger.Error(message)
}

// ...
```

デフォルトインスタンスをグローバルに格納しておき，グローバルインスタンス用のヘルパー関数も作っておく。こうすると以下のようにスッキリします。

```go
package users

import (
	// ...
	"fmt"
)

// ...

func NewStubRepository() Repository {
	return &stubRepository{
		users: map[int]*User{
			1: {ID: 1, Name: "Alice"},
			2: {ID: 2, Name: "Bob"},
		},
	}
}

// ...

type stubRepository struct {
	users  map[int]*User
}

func (r *stubRepository) FindByID(id int) *User {
	user, exists := r.users[id]
	if !exists {
		applog.Error("User not found")
		return nil
	}
	applog.Info(fmt.Sprintf("Fetched user with ID %d", id))
	return user
}

// ...

func NewService(repo Repository) *Service {
	return &Service{repo: repo}
}

type Service struct {
	repo Repository
}

func (s *Service) GetUserByID(id int) *User {
	applog.Info(fmt.Sprintf("Invoking repo.FindByID with %d", id))
	return s.repo.FindByID(id)
}
```

グローバル変数上等！割り切れ！…と言いたいところですが

### 問題点: インスタンスのスコープを限定できなくなってしまう

これはグローバルを選択したので当然ではあるんですが…

PHP であれば「1 プロセス 1 リクエスト」はお馴染みですね。これが許されているから， Laravel では雑にコンテナに `Request` オブジェクトが登録されていて，ゆえに `Request` を DI したりできるのです。ところが Go では *Goroutine* という仕組みによって， 1 プロセスの中で同時にたくさんのリクエストを捌くことができるようになっています。PHP でも [OpenSwoole](https://openswoole.com/), [RoadRunner](https://roadrunner.dev/) あたりは知名度は上がってきているかと思いますが，それらと似ています。

これが Logger とどう関連してくるかというと…例えばリクエストを追跡するために， **「トレース ID」** のようなものを全てのログに入れたいケース。複数のリクエストを同時に捌いているから，それらを関連付ける共通の ID が無いとログがぐちゃぐちゃになってしまいますよね。

```json lines
{"time":"2023-01-01T00:00:01Z","level":"info","message":"step-1","trace_id":"XXX"}
{"time":"2023-01-01T00:00:02Z","level":"info","message":"step-1","trace_id":"YYY"}
{"time":"2023-01-01T00:00:03Z","level":"info","message":"step-2","trace_id":"XXX"}
{"time":"2023-01-01T00:00:04Z","level":"info","message":"step-2","trace_id":"YYY"}
```

とはいえ毎回 *「使う側がトレース ID を渡してね」* は使いにくすぎて鬼畜すぎます。どうにかして「リクエストごとに共通な何か」を用意したいのですが，既存の仕組みではまだこれに対応できません。

### 問題点: テストの並列実行が困難

上述のリクエストスコープが実現できないのと同じように，グローバル依存はテストの並列実行可能性まで排除してしまいます。*「テストを高速化したい！」* ってときに大変辛くなるので，可能であれば回避したいところです。

# 推奨される実装パターン: コンテキストの利用

正解は **コンテキスト（[`context.Context`](https://pkg.go.dev/context)）** を使うことです。

## コンテキストとは？

以下の記事によくまとまっています。

https://zenn.dev/hsaki/books/golang-context

とはいえここで使わない情報も多いので，それらは省いた上で簡単に説明することにします。

### コンテキストは Goroutine セーフである

値の伝達手段としての目的が大きいので， Goroutine 間で安全に値を受け渡せるように設計されています。値の格納や取り出しに関して，自前でロックを取る必要性はありません。

### コンテキストはイミュータブルである

値は `<インスタンス>.Value(<キー>)` とすると取り出せる一方で， `<インスタンス>.SetValue(<キー>)` というものは用意されていません。イミュータブルであるおかげで，上述の Goroutine セーフである性質も実現しやすくなっていると考えていいと思います。

値を入れるために用意されているのは `context.WithValue(<インスタンス>, <キー>, <値>)` という関数で，これは合成したコンテキストを新しく返します。以下のように，コンテキストを提供するパッケージ単位で `WithContext` `FromContext` （コンテキストを利用して何かを実行する場合は `DoSomethingContext`） のような関数が作られることが非常に多いです。コンテキストを直接触ると `any` まみれになってしまうので，型安全性を担保するためにこのような関数が用意されます。

```go
package lang

import (
	"context"
)

type Lang string

const (
	Unknown Lang = ""
	English Lang = "English"
	Japanese Lang = "Japanese"
)

// 無名構造体を Opaque なキーとして使うことができる
type contextKey struct{}

func WithContext(ctx context.Context, lang Lang) context.Context {
	// 新しい値をセットして返す
	return context.WithValue(ctx, contextKey{}, lang)
}

func FromContext(ctx context.Context) Lang {
	// コンテキストに設定されていないときは取り出しがエラーになるが，エラーは _ への代入で無視している
	// その際 lang に入るのはゼロ値， 即ち値が空文字列である Unknown になる
	lang, _ := ctx.Value(contextKey{}).(Lang)
	// Unknown のときは English にフォールバック
	if lang == Unknown {
		lang = English
	}
	return lang
}
```

コンテキストを利用する側は以下のようなコードになります。

```go
package main

import (
	"context"
	"fmt"
	
	"example.com/myapp/lang"
)

func main() {
	ctx := context.Background()

	ctxWithEnglish := lang.WithContext(ctx, English)
	fmt.Println("Initial context value:", lang.FromContext(ctx))  // => "English" (フォールバック)
	fmt.Println("Context with English:", lang.FromContext(ctxWithEnglish))  // => "English"

	ctxWithJapanese := lang.WithContext(ctxWithEnglish, lang.Japanese)
	fmt.Println("Context with Japanese:", lang.FromContext(ctxWithJapanese))  // => "Japanese"
	fmt.Println("Context with English:", lang.FromContext(ctxWithEnglish))  // => "English" （イミュータブルであるため変化なし）
}
```

コンテキストに関する基礎的な知識としてはこれで十分です。

## コンテキストはとりあえず第1引数に受け取っておいて損はない

コンテキストは Go のアプリケーションだけに留まらず， Go の標準ライブラリなども全て含めて，非常に広範囲で利用される概念です。多くの関数は慣習として **「第1引数にコンテキストを受け取る」** という体裁に従います。

- [`http.Request.WithContext`](https://pkg.go.dev/net/http#Request.WithContext)
- [`sql.DB.QueryContext`](https://pkg.go.dev/database/sql#DB.QueryContext)
- [`sql.DB.ExecContext`](https://pkg.go.dev/database/sql#DB.ExecContext)
- [`os/exec.CommandContext`](https://pkg.go.dev/os/exec#CommandContext)
- [`net.Dialer.DialContext`](https://pkg.go.dev/net#Dialer.DialContext)
- [`net/http.Server.Shutdown`](https://pkg.go.dev/net/http#Server.Shutdown)

これに倣い，以下に該当する場所ではとりあえずコンテキストを受け取っておいて損はない，と考えていいと思います。

- **IO の伴う副作用を起こす**
  - **IO が伴う場所ではログの重要性も高いはず。コンテキストを用意すべき筆頭であると思われます**
- 拡張性が必要だが，その度に引数を追加するようなことをやりたくない
  - コンテキストであれば全て `context.Context` に詰め込めるので，引数を増やす必要がなく，拡張に強いです
    :::message
    但しオプション引数を実現したいだけであればコンテキストはオーバースペックで， **Functional Option Pattern** で十分である可能性があります。
    - [Goメモ-279 (Functional Option Patternのメモ) - いろいろ備忘録日記](https://devlights.hatenablog.com/entry/2022/12/19/073000)
    - [Go言語のFunctional Option Pattern - Qiita](https://qiita.com/weloan/items/56f1c7792088b5ede136)
    - [Go の 構造体定義から Functional Options Pattern のコードを自動生成する CLI ツールを作った](https://zenn.dev/ikechan0829/articles/foggo_generate_fop_cli)
    :::
- IO を伴うわけではないが，複雑な副作用を起こす
  - 臨機応変に判断

また仮に不必要に受け取ってしまったとしても，受け取っていないよりはずっと潰しがききます。明らかに不必要な場所は省いて，迷ったら受け取るぐらいでもいいんじゃないかと思います。

:::message
New Relic などの APM を利用している場合でも，セグメントを記録するためにコンテキストが必要です。

[budougumi0617/nrseg: Insert function segments into any function/method for Newrelic APM.](https://github.com/budougumi0617/nrseg)

上記のライブラリは全ての関数に APM 対応処理を入れてくれるツールですが，これも **コンテキストが第1引数にあることを前提** としています。
:::

## 何をコンテキストに載せるか？

### ロガーごとコンテキストに載せる

### デフォルト実装はグローバルに提供し，スコープを限定したい情報だけコンテキストから取得する

## エラーハンドリングの方法

### `error` を複値リターンに含めて返す

### `panic` する

### ほとんど副作用を起こさないデフォルト実装にフォールバックする
