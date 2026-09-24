---
title: "重複エラーログ根絶 Linter: errlogreturn"
emoji: "🪵"
type: "tech"
topics: ["go", "linter", "静的解析", "logging", "oss"]
published: false
publication_name: "yumemi_inc"
---

# はじめに

## 1 回の失敗が，ログには 3 回出てくる

障害調査でログを開いたら，同じエラーが 3 行並んでいた。そんな経験はありませんか？

```text
level=ERROR msg="failed to find user" id=42 err="sql: no rows in result set"
level=WARN  msg="load failed" err="find user 42: sql: no rows in result set"
level=ERROR msg="request failed" err="query: find user 42: sql: no rows in result set"
```

実際に失敗したのは 1 回だけです。リポジトリ層でログを出して返し，ユースケース層でもログを出して返し，最後にハンドラ層でまたログを出す。**レイヤーを通るたびにログが 1 行ずつ増えていきます。**

```go
func (r *UserRepository) Find(ctx context.Context, id int64) (*User, error) {
    var u User
    err := r.db.QueryRowContext(ctx, `SELECT id FROM users WHERE id = ?`, id).Scan(&u.ID)
    if err != nil {
        // ログに出して……
        slog.ErrorContext(ctx, "failed to find user", "id", id, "err", err)
        // さらに返す
        return nil, fmt.Errorf("find user %d: %w", id, err)
    }
    return &u, nil
}
```

これで何が困るかというと，

- **エラー件数が水増しされる。** アラートやダッシュボードの数字が，呼び出しの深さの分だけ膨らむ
- **どの行を見ればいいのか分からない。** 3 行のうちどれが本命なのか，読むたびに考えないといけない
- **お金がかかる。** ログ基盤の多くは量で課金されます

このあたりは Go の世界では昔から言われていることです。Dave Cheney 氏の有名な記事でも **「エラーのハンドリングは 1 回だけにしよう」** と書かれていて，ログに出すこともハンドリングの一種として扱われています。

https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully

ログに出したら，そこで終わりにする。返すなら，ログは呼び出し元に任せる。**ログか return か，どちらか一方だけ** にしましょう。

## AI エージェントも普通にやってくる

AI エージェントにエラー処理を書かせても，`return` の直前に `slog.Error` を 1 行足してくることがあります。世の中のコードにこの書き方がいくらでも転がっている以上，それを学習したエージェントが同じように書くのは自然な話です。

それに，目の前の関数だけを見ていれば，**ログを足しておくのは一見「安全」に見えます。** 呼び出し元でもうログを出しているかどうかは，その関数を見ているだけでは分かりません。コンパイルもテストも当然通りますし，レビューでも「丁寧に書いてあるな」くらいで流されがちです。

`CLAUDE.md` に「ログか return か，どちらか一方だけ」と書いておいても，守られるかどうかは **結局運次第** です。

それなら機械的にチェックしてしまえばいい，ということで作りました。

https://github.com/mpyw/errlogreturn

```bash
mise use "github:mpyw/errlogreturn@0.1.0"   # go install や go tool でも入ります
errlogreturn ./...
```

:::message alert
Go ではタブインデントが推奨されていますが，この記事では Web ブラウザ上での見やすさに配慮して半角スペースを採用しています。
:::

# 何を指摘するのか

冒頭のリポジトリにかけると，こうなります。

```console
$ errlogreturn ./...
app/repo.go:18:9: error is logged here and also returned at line 19; log it or return it, not both
app/repo.go:19:9:       returned here
```

:::message
記事中の出力はパスを短縮しています。実際には絶対パスで出力されます。
:::

**同じエラーを，ログにも出して return もしている** 箇所を指摘します。ログの行と `return` の行の両方が出るので，どちらかを消せば解消します。

どちらを消すかはケースバイケースですが，基本は **ラップして返すほうを残して，ログは一番上（HTTP ハンドラやミドルウェア，ジョブランナーなど）の 1 箇所にまとめる** のが素直だと思います。ラップするたびにエラーメッセージに文脈が足されていくので，一番上で 1 回出せば必要な情報は揃います。

```diff
 if err != nil {
-    slog.ErrorContext(ctx, "failed to find user", "id", id, "err", err)
     return nil, fmt.Errorf("find user %d: %w", id, err)
 }
```

## ラップしていても同じエラーとみなす

ログに出した値と返した値が，同じ変数である必要はありません。**元のエラーを材料にして作った値** であれば，同じエラーとして扱います。

| 形 | 例 |
|:---|:---|
| ラップ | `fmt.Errorf("load: %w", err)`，`errors.Join(err, other)` |
| フォーマット | `fmt.Errorf("load: %v", err)`，`err.Error()`。アンラップはできなくても，メッセージは呼び出し元に伝わる |
| 構造体に入れる | `&QueryError{Cause: err}` |
| ヘルパーを通す | 引数のエラーをそのまま戻り値に含めて返す関数 |
| ロガーのフィールド | `zap.Error(err)`，`slog.Any("err", err)`，`logger.With("err", err)` |

例えば，ログには `err.Error()` で文字列にして渡し，返すときは独自のエラー型に包んでいるケース。見た目は別物ですが，ちゃんと指摘されます。

```go
func Load(ctx context.Context, r *UserRepository, id int64) (*Profile, error) {
    u, err := r.Find(ctx, id)
    if err != nil {
        slog.WarnContext(ctx, "load failed", "err", err.Error())
        return nil, &UserQueryError{Cause: err}
    }
    return &Profile{User: u}, nil
}
```

```console
$ errlogreturn ./...
app/repo.go:33:9: error is logged here and also returned at line 34; log it or return it, not both
app/repo.go:34:9:       returned here
```

## 変数名ではなく中身を見ている

逆に，**同じ `err` という名前でも，中身が入れ替わっていれば別のエラー** として扱います。

```go
func Retry(ctx context.Context, r *UserRepository, id int64) error {
    _, err := r.Find(ctx, id)
    if err != nil {
        // 1 回目の失敗はログに出して……
        slog.WarnContext(ctx, "first attempt failed, retrying", "err", err)
        // 2 回目のエラーで上書きする
        _, err = r.Find(ctx, id)
    }
    // 返しているのは 2 回目のエラー。こちらはログに出ていない
    return err
}
```

これは指摘されません。ログに出したのは 1 回目の失敗で，返しているのは 2 回目の失敗だからです。

変数名だけで判定するような作りだと，ここで誤検知が出てしまいます。errlogreturn は値そのものを追っているので，上書きされた `err` は別物として区別できます。

# ヘルパー経由のログも拾う

実際のコードでは，ログは自前のヘルパーを通して出すことが多いですよね。

```go:logx/logx.go
package logx

// LogIfErr は err が nil でなければログに出す
func LogIfErr(ctx context.Context, err error) {
    if err != nil {
        slog.ErrorContext(ctx, "failed", "err", err)
    }
}

// LogUnlessCanceled はキャンセル以外のエラーだけログに出す
func LogUnlessCanceled(ctx context.Context, err error) {
    if !errors.Is(err, context.Canceled) {
        slog.ErrorContext(ctx, "failed", "err", err)
    }
}
```

```go:sync/sync.go
package sync

func Pull(ctx context.Context) error {
    err := fetch(ctx)
    logx.LogIfErr(ctx, err)
    return err
}

func Poll(ctx context.Context) error {
    err := fetch(ctx)
    logx.LogUnlessCanceled(ctx, err)
    return err
}
```

```console
$ errlogreturn ./...
sync/sync.go:11:5: error is logged by LogIfErr and also returned at line 12; log it or return it, not both
sync/sync.go:12:5:      returned here
```

**ヘルパーが別パッケージにあっても，設定なしで** 「これは引数をログに出す関数だ」と判断してくれます。

ポイントは，`Pull` は指摘されて `Poll` は指摘されていないところです。

| ヘルパーの中身 | 呼び出し側 |
|:---|:---|
| 引数を必ずログに出す | 指摘する |
| `nil` でなければログに出す | 指摘する |
| **それ以外の条件** のときだけログに出す | **指摘しない** |
| ログに出して，さらに返す | ヘルパー自身を指摘する。その戻り値を返しているだけの呼び出し側は指摘しない |

ロガー扱いするのは，**どの経路で `return` しても必ずログを出している** ヘルパーだけです。例外は「`nil` でなければ」という条件だけで，`nil` ならそもそもログに出すものがないので，これは「必ずログに出す」と同じとみなしています。

`LogUnlessCanceled` は，キャンセルのときはログを出しません。つまり `Poll` がキャンセルで失敗した場合，**呼び出し元に返さないと誰もそのエラーに気付けません。** なので，ここは指摘しないのが正解です。

# 対応しているロガー

| ロガー | 対象 | 対象外 |
|:---|:---|:---|
| `log` | `Print`，`Printf`，`Println`，`(*Logger).Output` | `Fatal*`，`Panic*` |
| `fmt` | `Print*`，`os.Stdout` / `os.Stderr` への `Fprint*` | それ以外の Writer への `Fprint*` |
| `log/slog` | `Info`，`Warn`，`Error` と各 `Context` 版 | `Debug`，`LevelInfo` 未満の `Log` |
| [zerolog](https://github.com/rs/zerolog) | `Info`，`Warn`，`Error`，`Err`，`Log` のイベントを `Msg` / `Msgf` / `MsgFunc` / `Send` で送ったもの | `Debug`，`Trace`，`Fatal`，`Panic`，送信されなかったイベント |
| [zap](https://github.com/uber-go/zap) | `Info`，`Warn`，`Error`，`DPanic` と Sugared 版 | `Debug`，`Fatal`，`Panic` |
| [logrus](https://github.com/sirupsen/logrus) | `Info`，`Warn`，`Warning`，`Error`，`Print` と `f` / `ln` 版 | `Debug`，`Trace`，`Fatal`，`Panic` |

**Debug レベルは対象外です。** リトライのたびに Debug ログを出すのはよくある書き方で，これはエラーを処理しているのではなく，経過を記録しているだけだからです。`Fatal` と `Panic` はそもそも戻ってこないので，後ろに `return` が来ることがありません。

これ以外の自前ロガーや，インタフェース越しに呼んでいるロガーは，ディレクティブで指定できます（後述）。

# 迷ったら指摘しない

errlogreturn は，**判断がつかないときは指摘しない** 方針で作っています。

見逃したときの損は，**ログが 1 行重複するだけ** です。一方で誤検知が続くと，**「この Linter の指摘は無視していいや」と思われてしまいます。** 人間にもエージェントにもです。こっちのほうがよっぽど痛いですよね。

なので，以下のようなケースは指摘しません。

| ケース | 理由 |
|:---|:---|
| ログと `return` が別々の分岐にある | 両方を通ることがない |
| `if verbose { log(err) }` の後に `if !verbose { return err }` | ログを出した場合はその `return` に来ない。一度通った条件は覚えている |
| 別のエラーを返している | `log(err); return ErrNotFound` は，ログに出したエラーを返していない |
| 別のエラーに変換している | 引数ではなく `ErrInternal` のような定義済みのエラーを返すヘルパーは，元のエラーを引き継いでいない |
| エラーの **判定結果** だけをログに出している | `err == nil` や `errors.Is(err, target)` で分かるのは失敗したかどうかで，エラーの中身ではない |
| ループのある周でログに出して，次の周で返している | 次の周のエラーは別物 |
| 生成されたファイル | 指摘されても直しようがない |

判定結果だけをログに出す書き方は，ミドルウェアでよく見かけますね。

```go
func IsMissing(ctx context.Context, r *UserRepository, id int64) error {
    _, err := r.Find(ctx, id)
    // 成功したかどうかだけを記録している。エラーの中身は出していない
    slog.InfoContext(ctx, "find done", "ok", err == nil)
    return err
}
```

これも指摘されません。

## 実際の OSS にかけて確認した

このあたりのルールは，ほとんどが実際のコードを見て決めたものです。リリース前に memos・pocketbase・caddy・traefik・jaeger など，ログを多用している OSS に片っ端からかけて，**出てきた指摘を 1 件ずつ全部読みました。** 誤検知を見つけるたびにルールを直していった結果，最後まで残ったのは **本当にログを出して return しているものか，標準ライブラリ内のわざとそうしているトレース用のラッパーのどちらか** だけになりました。

# 指摘を抑える・ロガーを指定する

## `//errlogreturn:ignore`

どうしても両方やりたい場合は，指摘された行かその 1 行上にディレクティブを書きます。後ろには好きなことを書けるので，**なぜ両方必要なのかを書いておきましょう。**

```go
err := dial()
//errlogreturn:ignore 呼び出し元はこのエラーをトレースにしか使わない
slog.Error("open failed",
    "err", err)
return err
```

`//go:` ディレクティブと同じで，スラッシュの後にスペースを入れると効きません。`// errlogreturn:ignore` はただのコメント扱いです。

:::message
**何も抑えていない `//errlogreturn:ignore` は指摘されます。** コードを直した後に消し忘れた ignore が残り続けることはありません。
:::

## `//errlogreturn:sink`

インタフェースのメソッドや，中身を解析できないロガーは，doc コメントに `//errlogreturn:sink` を書いて **「これは引数を全部ログに出す関数だ」** と教えてあげます。

```go
type Reporter interface {
    //errlogreturn:sink
    Report(msg string, args ...any)
}

func Upload(r Reporter) error {
    err := send()
    r.Report("upload failed", "err", err)
    return err
}
```

```console
$ errlogreturn ./...
billing/billing.go:12:5: error is logged here and also returned at line 13; log it or return it, not both
billing/billing.go:13:5:        returned here
```

自分で手を入れられないサードパーティのロガーは，`-sinks` フラグで指定します。

```bash
errlogreturn -sinks 'example.com/telemetry.Send,(example.com/telemetry.Client).Capture' ./...

# go vet 経由でも同じように渡せます
go vet -vettool=$(which errlogreturn) -sinks 'example.com/telemetry.Send' ./...
```

# 導入

| 方法 | コマンド |
|:---|:---|
| **[mise](https://mise.jdx.dev/)** （推奨） | `mise use "github:mpyw/errlogreturn@0.1.0"` |
| `go tool` | `go get -tool github.com/mpyw/errlogreturn/cmd/errlogreturn@latest` |
| `go install` | `go install github.com/mpyw/errlogreturn/cmd/errlogreturn@latest` |

ビルドには Go 1.27 以上が必要ですが，解析するコードの Go バージョンは問いません。

## 大きなモジュールでは `go vet` 経由で

```bash
go vet -vettool=$(which errlogreturn) ./...
```

別パッケージのヘルパーを判断するために依存パッケージもまるごと解析するので，単体で実行すると traefik では **メモリが 19GB** まで膨らみました。`go vet` 経由ならパッケージごとに別プロセスで解析してキャッシュも効くので，**2.7GB**，2 回目以降は **0.2GB** で済みます。

:::message alert
CI/CD パイプラインでは `@latest` ではなく `@v0.1.0` のようにバージョンを固定してください。サプライチェーン攻撃への備えです。
:::

# 制約

「迷ったら指摘しない」方針なので，以下のようなケースは拾えません。

- **インタフェースや関数値を通した呼び出し。** 呼び出し先が分からないので，ログは出さないものとして扱う。ログを出すものなら `//errlogreturn:sink` で指定する
- **チャネルで送ったエラーや，グローバル変数に入れて別の関数で読むエラー。** そこまでは追いかけない
- **再帰するヘルパー。** 自分自身の呼び出しはログを出さないものとして扱う

どれも見逃す側に倒しています。見逃しても，ログが 1 行重複するだけです。

# まとめ

- **1 回の失敗につきログは 1 回。** ログに出して返すと，レイヤーの数だけ同じエラーが並び，件数も読みやすさもコストも悪くなる
- **AI エージェントもこの書き方をしがち。** 目の前の関数からは呼び出し元のログが見えないので，足しておくのが安全に見えてしまう。`CLAUDE.md` に書いても止められるとは限らない
- **errlogreturn は，同じエラーをログにも出して return もしている箇所を指摘する。** ラップ・フォーマット・構造体に入れる・ヘルパーを通す，どれでも拾うし，上書きされた `err` は別物として区別する
- **大きなモジュールでは `go vet -vettool` 経由で。** メモリ使用量がまるで違う

https://github.com/mpyw/errlogreturn

使ってみて「これ誤検知じゃない？」というケースがあれば，ぜひ Issue やコメント欄で教えてください。**誤検知の報告がいちばん助かります。**
