---
title: "New Relic Go Agent 完全理解・実践導入ガイド"
emoji: "☠️"
type: "tech"
topics: ["go", "newrelic", "telemetry", "apm", "logging"]
published: true
---

:::message
Go ではタブインデントが推奨されていますが，この記事では Web ブラウザ上での見やすさに配慮して半角スペースを採用しています。
:::

:::message
この記事は Forkwell の~~普段は絶対読まない~~メルマガで配信されていた

**[おい、テックブログを書け - Speaker Deck](https://speakerdeck.com/nwiizo/oi-tetukuburoguwoshu-ke)**

が目に留まり，どうしても頭から離れなくなってしまったので，発散してスッキリするために~~仕方なく~~書きました。いつから俺の筆はこんなに重くなってしまったんだ…
:::

# ご挨拶

この記事は [Go - Qiita Advent Calendar 2025](https://qiita.com/advent-calendar/2025/go) (Series 2) の 14 日目の記事です。

ごぶさたしております。現在の近況を簡単にご報告させてください。

- 勤務していた会社が消滅しました。人生 2 回目です。
  - [1回目](https://dmm-corp.com/press/corporate/388/)
  - [2回目](https://www.itmedia.co.jp/news/articles/2510/28/news108.html)
- [𝕏 アカウント](https://x.com/mpyw)が凍結されました。
  - ~~不正な凍結代行業が横行する恐ろしいプラットフォームです~~。迂闊な~~政治的~~発言はやめましょう
  - ~~フォロワー 9000 人吹っ飛んだのは別に悲しくないけど~~，情報アンテナ感度が下がったのが結構痛いです
  - みなさんもっと [Bluesky](https://bsky.app) に来てください

https://bsky.app/profile/mpyw.bsky.social

最近は Go をメインに書いていて，ほとんど PHP を書いていません。以前は Go の冗長すぎる文法が嫌いだったのですが，[samber/lo](https://pkg.go.dev/github.com/samber/lo) を使えば大幅に苦痛は軽減されますし，生成 AI 時代には Go ぐらいのシンプルな文法のほうが可読性・再現性ともに高くて，逆にアドバンテージがあると感じています。

（ちなみに PHP は 8.5 から [Pipeline Operator](https://php.watch/versions/8.5/pipe-operator) という，書き方が人によって激しく分かれる思い切った新機能を導入しましたが，果たしてどこに向かうのやら…）

さて，最近 Go 案件で New Relic 導入を手伝う機会がありました。過去に PHP + New Relic の組み合わせは結構調べたのですが，今回 Go + New Relic でしっかり構成するのは初めてでした。かなりハマりポイントも多かったので，そのあたりを中心に解説していきたいと思います。

↓過去の調査
https://speakerdeck.com/mpyw/tui-ce-suruna-ji-ce-seyo-new-relic-x-laravel-shi-jian

# New Relic とは？

New Relic は **オブザーバビリティ・プラットフォーム** の 1 つです。アプリケーションやインフラの監視・分析を統合的に行うことができる SaaS 製品です。監視対象のマシンやプログラムから自動的にデータを収集し，分析する **テレメトリ** という機能を持っています。現在業界では OpenTelemetry (OTel) が標準規格として広まりつつありますが， New Relic は OTel からの取り込みをサポートしつつも，自社の技術に最適化された独自のエージェントや SDK も引き続き提供しています。

# Go + New Relic におけるトレーシングおよびその手法の比較

テレメトリ機能のうち， **トレーシング（トレース計測）** と呼ばれるものに着目しましょう。これは，1 つのトランザクション（例: 1 つの HTTP リクエスト処理）の中で，さらに細かい処理単位（例: DB クエリ，外部 API 呼び出し，あるいは単なる関数呼び出し）を計測する機能です。そのうちの 1 単位を **セグメント（より一般的には「スパン」）** と呼びます。これを計測する方法を，以下の 2 軸で比較してみます。

- 通信境界計装: HTTP ハンドラや gRPC サーバーなど，**プロセス間通信の境界** における計装
- 全関数計装: [`context.Context`](https://pkg.go.dev/context#Context) を第 1 引数として受け取る関数すべてに対する計装

一般的に，通信境界計装はミドルウェアやラッパーを用意するだけで済むことが多く，導入が容易です。一方，全関数計装はプログラミング言語によって大きくアプローチが異なる部分です。 Go は Executable Binary を直接生成する静的コンパイル言語であり，実行時に仮想マシンやインタプリタを介さないため，動的な干渉が困難になるという特性があります。 また全関数計装は，実現手段次第ではそこそこのオーバーヘッドが発生するため，導入の際にはパフォーマンス面での考慮も必要です。

:::message
実際には全てのトランザクションが記録されるわけではなく，一定割合で **サンプリング** される形になるため，直感的な印象よりは影響が小さく収まることも期待できます。
:::

|                                |              通信境界計装               |        全関数計装        |
|:-------------------------------|:---------------------------------:|:-------------------:|
| **[A] 計装用コードを手作業で記載して Git 管理** |   😁<br>**ミドルウェアやラッパーを用意するだけ**    |      🤮<br>虱潰し      |
| **[B] 計装用コードを自動生成して Git 管理**   |                 ー                 |   😃<br>**比較的簡単**   |
| [C] ビルド時に透過的にコード変換ツールで干渉       |       😁<br>ビルドコマンドを変更するだけ        |      🤮<br>虱潰し      |
| **[D] Linux カーネルに干渉 (eBPF)**   | 😁<br>**インストールするだけ**<br>（但し環境は選ぶ） | ☠️<br>技術的には可能だが無理ゲー |

:::details 参考: 他の言語でよく使われる全関数計装手法

主要言語の中で，クラスを動的に変更することが難しい言語の例としては， Java と PHP が挙げられます。これらの言語では，以下のような仕組みでランタイムに干渉し，全関数計装を実現しています。

| 言語   | アプローチの分類                               | 技術的な仕組み                                                                        |
|:-----|:---------------------------------------|:-------------------------------------------------------------------------------|
| Java | バイトコード操作<br>(Bytecode Instrumentation) | JVM 起動時 `-javaagent` オプションでクラスローダーに介入し，バイトコードが書かれた `*.class` ファイルの内容を動的に書き換える。 |
| PHP  | C 拡張モジュール<br>(Zend Engine Hook)        | C 言語で書かれた拡張モジュールとしてロードされ， Zend Engine 内部の関数ポインタをフックする。                         |

（Python や Ruby はモンキーパッチが可能なため割愛）
:::

## [A] 計測用コードを手作業で記載して Git 管理

### 通信境界計装: 😁ミドルウェアやラッパーを用意するだけ

数行挿入するだけで済むことが多く，導入はとても簡単です。 [echo](https://pkg.go.dev/github.com/labstack/echo) における New Relic ミドルウェアの例を示します。

https://pkg.go.dev/github.com/newrelic/go-agent/_integrations/nrecho

```go
e := echo.New()
e.Use(nrecho.Middleware(app))
```

### 全関数計装: 🤮虱潰し

最も原始的な方法です。アプリケーションのコードに直接計測用のコードを埋め込みます。

```go
func DoSomething(ctx context.Context) {
    // ↓ 手動で挿入
    defer newrelic.FromContext(ctx).StartSegment("DoSomething").End()

    // ...
}
```

:::details defer 文の書き方に違和感がある人のための補足
メソッドチェインは通常の関数呼び出しの第 1 引数を省略したシンタックスシュガーであるため，上記の `defer` 文は以下と等価です。

```go
defer (*newrelic.Segment).End(
    (*newrelic.Transaction).StartSegment(
        newrelic.FromContext(ctx),
        "DoSomething",
    ),
)
```

また `defer` 文はルートの関数コールだけを遅延させ，**引数の評価は即座に行います**。そのため，遅延されるのは `End` メソッドの呼び出しだけであり，`StartSegment` の呼び出しは即座に実行されます。
:::

:::message
[`runtime.Caller`](https://pkg.go.dev/runtime#Caller) を使って関数名やファイル名を自動取得する方法もありますが，補助手段に留めるべきです。使用箇所が広範囲に及ぶと，パフォーマンスに悪影響を及ぼす可能性があります。
:::

柔軟性は非常に高いですが，全てが開発者に委ねられます。保守し切るのは茨の道でしょう。

## [B] 計測用コードを自動生成して Git 管理

### 全関数計装: 😃比較的簡単

原始的な方法をコード生成アプローチで改善したものです。実用性を考えると，一番推奨される方法です。コード生成自体は Go のカルチャーともフィットしていると思います。

https://pkg.go.dev/github.com/budougumi0617/nrseg

```go
func DoSomething(ctx context.Context) {
    // ↓ 自動生成ツールで挿入
    defer newrelic.FromContext(ctx).StartSegment("DoSomething").End()

    // ...
}
```

上記のように生成されるコードが先頭 1 行で収まる程度であれば上出来でしょう。但し， New Relic にアプリケーションコードを直接依存させず，自前で用意したパッケージで具象を隠蔽することを考える…とか欲張りだすと，基本的には自作することになるかなと思います。

:::message
CI で更新適用後， `git diff --exit-code` で更新漏れを検出しましょう。
:::

:::details 一応…
標準ライブラリや著名ライブラリのプロトコル通信に限定し，自動計装コードを挿入してくれるツールがあります。一応 New Relic 公式のツールですが，スター数が少なすぎる…きっとこれは永遠の Experimental…

https://pkg.go.dev/github.com/newrelic/go-easy-instrumentation

本音を言うと，通信境界だったらミドルウェアが普通用意されているものだし，ここを狙い撃ちにしたコード生成はかなり悪手だと思います。
:::

> 💬 *率直に言うと: Go に一番適した方法ではあるが，ツールの融通が効かなくて自作しがち。私はしました…*

## [C] ビルド時に透過的にコード変換ツールで干渉

### 通信境界計装: 😁ビルドコマンドを変更するだけ

ビルド時にコードを解析・変換して，計測用のコードを自動挿入します。技術的には `go build` の `-toolexec` オプションを使って，コンパイル処理の前段に独自ツールを挟み込む形で実現されます。**アプリケーションコードが汚染されない**のが最大の利点です。

…というのが一般論ではあるんですが， New Relic 公式はこのようなアプローチを提供していません。一応 OpenTelemetry ベースのツールを利用して収集し，それを New Relic に送信する形なら実現できるとは思います。

↓ Alibaba 社の LoongSuite Go Agent。こちらは OpenTelemetry ベースです。
https://pkg.go.dev/github.com/alibaba/loongsuite-go-agent

↓ 参考: Datadog 社の Orchestrion
https://pkg.go.dev/github.com/DataDog/orchestrion

### 全関数計装: 🤮虱潰し

- Datadog については， `//dd:span` というコメントを書いておくだけで，上記の Orchestrion が自動的に計装してくれます。
- LoongSuite の場合は自前でコードを書く必要があります。

結局はこの部分は虱潰しになりますが，要求されるコメントまたはコードについて，それを生成する [B] のアプローチを組み合わせれば，幾分かマシにはなるはずです。

## [D] Linux カーネルに干渉 (eBPF)

[eBPF (extended Berkeley Packet Filter)](https://en.wikipedia.org/wiki/EBPF) を使い，カーネルレベルでシステムコールやネットワークトラフィックを監視します。

*「言語に VM がないの？ならば OS で干渉できればいいじゃん！」*

という発想で発展してきた技術のようです。 [`context.Context`](https://pkg.go.dev/context#Context) の引き回しが必須，計装用コードの注入も必須，という Go 開発者にとっては夢のような手法ですね。 Telemetry ライブラリを作る側はものすごく大変そうですが…

:::message alert
2025年12月現在， Fargate や Cloud Run などコンテナの抽象度が高い環境では eBPF は利用できないようです。
:::

### 通信境界計装: 😁インストールするだけ

↓ OpenTelemetry の eBPF ベースのプロジェクト
https://pkg.go.dev/github.com/open-telemetry/opentelemetry-go-instrumentation

↓ 上記のアーキテクチャ解説
https://opentelemetry.io/docs/zero-code/obi/

↓ 最近注目されている Grafana Beyla
（OSS だがマネージドサービスは有料）
https://grafana.com/ja/oss/beyla-ebpf/

対応範囲が一般的なものに限定されていれば， OSS コミュニティの先人の知恵を借りるだけで目的は達成できそうです。

### 全関数計装: ☠️技術的には可能だが無理ゲー

OSS コミュニティがスコープを限定して慎重にやっていることを，プロジェクト内のあらゆる関数を狙い撃ちにして…という話になりますが，どう考えても無理ゲーですね。とはいえ，野望を捨てきれない人向けに，参考情報を示しておきます。低レイヤー技術が好きな人は食いつきたくなる話題かもしれません。

https://mackerel.io/ja/blog/entry/tech/auto-instrument-with-golang-ebpf

# New Relic Go Agent を構成する主要な要素

ここからより実践的な話をしていきます。 OpenTelemetry ベースの手法もありますが，ここでは New Relic Go Agent を直接利用する方法に絞って説明します。まずエージェントの構成を理解する上で押さえておくべき 3 つの中心的な型があります。大雑把に役割およびライフサイクルを押さえておきましょう。

```
newrelic.Application (アプリケーション全体)
└── newrelic.Transaction (1 つのリクエスト処理)
    └── newrelic.Segment (関数呼び出し)
```

## [`newrelic.Application`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/newrelic#Application)

エージェントの基底構造体です。アプリケーション起動時に **1 回だけ作成**され，**グローバルに共有**されます。 Goroutine 間で安全に共有できます。

```go
// グローバル変数として保持（推奨）
var App *newrelic.Application

// アプリケーション起動時に main 関数から呼ぶ想定
func BootstrapNewRelic() error {
    var err error

    // アプリケーション起動時に 1 回だけ実行
    App, err = newrelic.NewApplication(
        // ... 設定
    )

    // 設定に不備があった場合のエラーハンドリング
    if err != nil {
        return err
    }
}
```

:::message
これ以降ではグローバル変数 `App` が [`*newrelic.Application`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/newrelic#Application) を指しているものとします。
:::

## [`newrelic.Transaction`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/newrelic#Transaction)

**1 つのトランザクション（処理の単位）を表す構造体** です。HTTP リクエストの処理やバッチ処理の実行など，計測したい処理の開始から終了までを追跡します。これがコンテキストに埋め込まれ，引き回されることを想定した設計です。

:::message alert
トランザクションは Goroutine 間で共有してはなりません。もし新しい Goroutine を起動する場合は， [`(*newrelic.Transaction).NewGoroutine()`](https://pkg.go.dev/github.com/newrelic/go-agent#Transaction.NewGoroutine) メソッドで派生トランザクションを作る必要があります。
（詳しくは後述します）
:::

```go
// HTTP ハンドラの例
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    // トランザクションを開始
    txn := App.StartTransaction("HandleRequest")
    defer txn.End() // 処理終了時に必ず呼ぶ

    // トランザクションを Context に埋め込んで引き回せるようにする（強く推奨）
    ctx := newrelic.NewContext(r.Context(), txn)

    // 上記の ctx を引き回して後続処理を実行
    // ...
}
```

```go
// 取り出すとき
txn := newrelic.FromContext(ctx)
```

## [`newrelic.Segment`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/newrelic#Segment)

**トランザクション内の 1 つのセグメント（処理の単位）を表す構造体** です。DB クエリ，外部 API 呼び出し，関数の実行など，トランザクション内の細かい処理を計測します。トランザクションの子要素として作成されます。

また種類によって専用の型があります：
- [`newrelic.Segment`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/newrelic#Segment): 汎用
- [`newrelic.DatastoreSegment`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/newrelic#DatastoreSegment): DB クエリ専用
- [`newrelic.ExternalSegment`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/newrelic#ExternalSegment): 外部 API 呼び出し専用

```go
func DoSomething(ctx context.Context) {
    // ↓ 手動で挿入
    defer newrelic.FromContext(ctx).StartSegment("DoSomething").End()

    // ...
}
```

# Logs in Context: ログのトレースへの関連付け

また併せて押さえておきたいのが **[Logs in Context](https://docs.newrelic.com/docs/logs/logs-context/logs-in-context/)** と呼ばれる機能です。 New Relic は APM でのトレーシングとは別の機能として，伝統的なログ管理機能も提供しています。本来用途が異なるものではあるけれども，片方を調査中にもう片方の情報も欲しくなることはよくありますよね。従来この 2 つは別の概念だったのですが， Logs in Context という機能が登場してからは，ログとトレースを関連付けて扱えるようになりました。

## トレースとログの違い

トレースとログの違いを，以下の表にまとめます。一部 New Relic 固有の用語が含まれていますが，概念としては他のオブザーバビリティ・プラットフォームでも共通する部分が多いです。

|                       | トレース                                                                                         | ログ                                                                                                                                                                                                                                                                                                                                                 |
|:----------------------|:---------------------------------------------------------------------------------------------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **目的**                | 連続した処理の計測と追跡                                                                                 | イベントの記録                                                                                                                                                                                                                                                                                                                                            |
| **単位同士の関係**           | 親子関係を持つ階層構造                                                                                  | 単一で独立                                                                                                                                                                                                                                                                                                                                              |
| **記録箇所**              | 多い                                                                                           | 少ない                                                                                                                                                                                                                                                                                                                                                |
| **サンプリング**            | される場合が多い                                                                                     | されない場合が多い<br>**（結果として総量はトレースより圧倒的に多くなる）**                                                                                                                                                                                                                                                                                                          |
| **単位あたりのサイズ**         | 小さい                                                                                          | 大きい                                                                                                                                                                                                                                                                                                                                                |
| **保存期間**              | 短期                                                                                           | 長期                                                                                                                                                                                                                                                                                                                                                 |
| **利用シナリオ**            | 障害検知<br>パフォーマンスチューニング                                                                        | 障害分析<br>運用分析                                                                                                                                                                                                                                                                                                                                       |
| **代表的なフィールド**         | タイムスタンプ<br>**トレース（トランザクション） ID**<br>**スパン（セグメント） ID**<br>経過時間                                | タイムスタンプ<br>ログレベル<br>メッセージ                                                                                                                                                                                                                                                                                                                          |
| **New Relic 上での機能分類** | [APM](https://docs.newrelic.com/jp/docs/apm/new-relic-apm/getting-started/introduction-apm/) | [Logs](https://docs.newrelic.com/jp/docs/logs/get-started/get-started-log-management/)                                                                                                                                                                                                                                                             |
| **New Relic への転送方法**  | アプリケーションコンテナ内のエージェントが直接 New Relic に送信                                                        | **[通常]**<br>アプリケーションコンテナの標準出力またはファイル出力をサイドカーコンテナが収集して送信（するように構築する）<br><br>**[エージェントの Log Forwarding 機能を有効化し，かつログ出力先の [`io.Writer`](https://pkg.go.dev/io#Writer) が [`nrwriter.LogWriter`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/integrations/logcontext-v2/nrwriter#LogWriter) に設定されている場合]**<br>アプリケーションコンテナ内のエージェントが直接 New Relic に送信 |

### 補足: Log Forwarding 機能とは？

アプリケーションコンテナ内で発生したログを，エージェントが直接 New Relic に送信する機能です。通常はアプリケーションコンテナの標準出力やファイル出力をサイドカーコンテナが収集して送信する必要がありますが， New Relic Go Agent がロガーに接続済みである場合は，そのまま発見したログを転送してもらうことができます。

:::message alert
**Log Forwarding はコンテナの標準出力を拾うわけではありません**。 [`nrwriter.LogWriter`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/integrations/logcontext-v2/nrwriter#LogWriter) にログが書き込まれる瞬間にイベントとして発生した **[`newrelic.LogData`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/newrelic#LogData)** という構造体に記録されたものを転送するようです。

これだけ聞くと「テキストとしてシリアライズされる前の元の構造体情報」を拾えそうに思うかもしれませんが，実際は厳しいです。実装を見てもらうと分かりますが， **[`io.Writer`](https://pkg.go.dev/io#Writer) インタフェースで引数から入ってくるのは `[]byte` 型のデータであるため，これをパースして復元する** というものすごい泥臭い処理が書かれています。それもあってかログドライバーによって品質にバラつきがあり，もし確実に書いた通りにログを転送したいのであれば Log Forwarding 機能は使わないほうが無難かもしれません。どういうわけか 2025 年 12 月現在 [`zerolog` だけカスタム属性対応が未実装](https://github.com/newrelic/go-agent/blob/e9f24662e50b29c3e8eeba5a299780edcda9cc17/v3/integrations/logcontext-v2/zerologWriter/zerolog-writer.go#L82)ですね…
:::

## ログをトレースに関連付けるための Enrichment 処理

ログをトレースに関連付けるためには，上記の「代表的なフィールド」に記載されている，

- トレース（トランザクション） ID
- スパン（セグメント） ID

この 2 つの情報をログ側にも付与することによって実現されます。この処理を **Enrichment (装飾)** と呼びます。

```diff
 {
     "timestamp": "2024-01-01T12:00:00Z",
     "level": "ERROR",
     "message": "Something went wrong",
+    "trace.id": "0123456789abcdef",
+    "span.id": "abcdef0123456789"
 }
```

New Relic に取り込まれた後のログは上記のような形になるのですが，実際には [`nrwriter.LogWriter`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/integrations/logcontext-v2/nrwriter#LogWriter) は JSON 形式以外のテキスト形式のログにも対応できるよう， JSON 形式に依存しないフォーマットで装飾しています。

### 装飾フォーマット

```markdown
<オリジナルのペイロード> NR-LINKING|<エンティティ GUID>|<ホスト名>|<トレース ID>|<スパン ID>|<アプリケーション名>|
```

### テキスト形式のログが装飾された場合

```text
2024-01-01T10:00:00Z INFO test log NR-LINKING|guid123|host|trace456|span789|my-example-app|
```

### JSON (JSONL) 形式のログが装飾された場合

```json
{"level":"info","time":"2024-01-01T10:00:00Z","message":"test log"} NR-LINKING|guid123|host|trace456|span789|my-example-app|
```

JSONL 形式が New Relic 固有の事情によってぶっ壊されてるので注意してください。 New Relic 以外にパースさせる場合は大問題ですね…

## New Relic Go Agent のロガーへの接続

New Relic Go Agent がログを取り扱うにあたり，お使いのロガーを New Relic Go Agent ([`*newrelic.Application`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/newrelic#Application)) に接続する必要があります。ここでいう接続とは，ロガーの書き込み先の [`io.Writer`](https://pkg.go.dev/io#Writer) インターフェースに， New Relic Go Agent が提供する [`nrwriter.LogWriter`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/integrations/logcontext-v2/nrwriter#LogWriter) を設定することを指します。これにより， New Relic Go Agent がログ出力を検知できるようになります。

:::details 標準 log パッケージの場合
```go
import (
    "log"
    "os"

    "github.com/newrelic/go-agent/v3/integrations/logcontext-v2/nrwriter"
)

func InitGlobalLogger() {
    // 標準 log パッケージでグローバルロガーに設定
    log.SetOutput(nrwriter.New(os.Stdout, App))
}
```
:::

:::details 標準 slog パッケージの場合
```go
import (
    "log/slog"
    "os"

    "github.com/newrelic/go-agent/v3/integrations/logcontext-v2/nrslog"
)

func InitGlobalLogger() {
    // 標準 slog パッケージでグローバルロガーに設定
    slog.SetDefault(slog.New(nrslog.JSONHandler(App, os.Stdout, &slog.HandlerOptions{})))
}
```
:::

:::details zerolog パッケージの場合
```go
import (
    "os"

    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
    "github.com/newrelic/go-agent/v3/integrations/logcontext-v2/zerologWriter"
)

func InitGlobalLogger() {
    // zerolog/log パッケージでグローバルロガーに設定
    // (zerologWriter は nrwriter を内包している）
    log.Logger = log.Logger.Output(zerologWriter.New(os.Stdout, App))

    // zerolog はグローバルロガーとデフォルトコンテキストロガーをバラバラで管理しているため，
    // コンテキストロガーのフォールバック先もグローバルロガーに設定しておく
    zerolog.DefaultContextLogger = &log.Logger
}
```
:::

## トランザクション内でのロガーの使用

実際にトレース情報をログに自動的に埋め込むためには，**トランザクションに紐づいたロガー** を使用する必要があります。そのままではグローバルロガーが使われてしまい，トランザクションやセグメントの情報が紐付けられないためです。

以下に各ロガーライブラリごとの例を示します。ここでは意図的に， HTTP 文脈に依存しない普遍的な書き方をしています。以下， `GetContextEnrichedLogger(ctx)` でトランザクションに紐づいたロガーを取得できるように書いてみます。実際にはもう少しいい感じのネーミングにしてくださいね…！

:::details 標準 log パッケージの場合
```go
import (
    "log"
    "os"

    "github.com/newrelic/go-agent/v3/integrations/logcontext-v2/nrwriter"
)

// newrelic.FromContext(ctx) でトランザクションが取得できる前提
func GetContextEnrichedLogger(ctx context.Context) *log.Logger {
    writer := nrwriter.New(os.Stdout, App)
    writer = writer.WithContext(ctx)

    return log.New(writer, "", log.LstdFlags)
}
```
:::

:::details 標準 slog パッケージの場合
```go
import (
    "log/slog"

    "github.com/newrelic/go-agent/v3/integrations/logcontext-v2/nrslog"
)

// newrelic.FromContext(ctx) でトランザクションが取得できる前提
func GetContextEnrichedLogger(ctx context.Context) *slog.Logger {
    return nrslog.WithContext(ctx, slog.Default())
}
```

[`slog`](https://pkg.go.dev/log/slog) は元のロガーから既に設定済みの [`nrslog.NRHandler`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/integrations/logcontext-v2/nrslog) を取り出して一部変更できるようになっているため，他のパターンよりもややシンプルに記述できています。
:::

:::details zerolog パッケージの場合
他のロガーライブラリと同様に，グローバルロガーから派生する場合は以下のようになります。

```go
import (
    "context"

    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
    "github.com/newrelic/go-agent/v3/integrations/logcontext-v2/zerologWriter"
)

// newrelic.FromContext(ctx) でトランザクションが取得できる前提
func GetContextEnrichedLogger(ctx context.Context) zerolog.Logger {
    writer := zerologWriter.New(os.Stdout, App)
    writer = writer.WithContext(ctx)

    // グローバルロガーから Output 設定だけを変更した新しいロガーを返す
    return log.Output(writer)
}
```

但し，コンテキストロガーを常に **[`log.Ctx(ctx)`](https://pkg.go.dev/github.com/rs/zerolog/log#Ctx)** で取得できるようにしておくのが [`zerolog`](https://pkg.go.dev/github.com/rs/zerolog) のお作法ゆえ，以下のように実装することを推奨します。

```go
import (
    "context"

    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
    "github.com/newrelic/go-agent/v3/integrations/logcontext-v2/zerologWriter"
)

// newrelic.FromContext(ctx) でトランザクションが取得できる前提
func WithContextEnrichment(ctx context.Context) context.Context {
    writer := zerologWriter.New(os.Stdout, App)
    writer = writer.WithContext(ctx)

    // コンテキストロガーから Output 設定だけを変更した新しいロガーを作成し，新たにコンテキストに上乗せする
    return log.Ctx(ctx).Output(writer).WithContext(ctx)
}
```
:::

他のライブラリを用いる場合でも， [`zerolog`](https://pkg.go.dev/github.com/rs/zerolog) の設計思想は非常に参考になるとは思います。やはりコンテキスト経由は正義…！

https://zenn.dev/mpyw/articles/go-dont-inject-logger

（これで昔痛い目見たなぁ…）

# トレースに付帯するエラー

トレースにはエラー情報を付帯させることができます。トレースのエラーは APM ダッシュボード上で一目で確認できるため，障害対応などに非常に役立つでしょう。以下では， New Relic Go Agent におけるエラー情報の取り扱いについて説明します。

:::message
**「エラーが付帯したトレース」** と **「エラーレベルが設定されたログ」** は完全に別物です。とはいえ，使い分けには大いに悩むと思います。 Logs in Context が利用されている場合は両者の距離が近づくため，余計に悩みますね。アプリケーションの要件次第ではありますが，一般的には

- トランザクションあたりのエラー報告は **トップレベルの 1 回のみ**
  - この報告はログと重複しても構わない。むしろログのほうが保存期間が長い特性上，重複して記録したほうがよい
- 内部で起こった細かいエラーは，回数制限なく必要に応じてログに記録する

という使い分けが理想的だと思われます。
:::

## スタックトレース対応

Go では標準の [`errors.New()`](https://pkg.go.dev/errors#New) や [`fmt.Errorf()`](https://pkg.go.dev/fmt#Errorf) で生成したエラーにはスタックトレース情報が含まれません。これはパフォーマンスを重視した設計思想によるものですが，I/O 処理が多い Web アプリケーションにおいてはスタックトレースによるオーバーヘッドは些細で，それよりもデバッグ性を重視したい場面が多いと思われます。また New Relic APM では，エラーのスタックトレースを APM ダッシュボード上で確認する機能もあります。また APM だけでなく，シンプルにログを確認する場面でも役に立つことが多いです。

https://pkg.go.dev/github.com/cockroachdb/errors

または

https://pkg.go.dev/github.com/rotisserie/eris

この辺りのライブラリはぜひ導入し，標準の [`errors`](https://pkg.go.dev/errors) パッケージは使わないようにしておきましょう。

## エラーを通知する方法

エラーが通知されるフローは以下の 3 通りあります。

### [A] 明示的なエラー通知

以下の 2 つのメソッドを使って，明示的にエラーを通知します。

| 用途   | メソッド                                                                                                                                                                                |
|------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 異常系  | [`(*newrelic.Transaction).NoticeError(err error)`](https://github.com/newrelic/go-agent/blob/e9f24662e50b29c3e8eeba5a299780edcda9cc17/v3/newrelic/transaction.go#L107-L137)         |
| 準正常系 | [`(*newrelic.Transaction).NoticeExpectedError(err error)`](https://github.com/newrelic/go-agent/blob/e9f24662e50b29c3e8eeba5a299780edcda9cc17/v3/newrelic/transaction.go#L139-L171) |

ここで通知されるエラーは [`errDataFromError`](https://github.com/newrelic/go-agent/blob/e9f24662e50b29c3e8eeba5a299780edcda9cc17/v3/newrelic/internal_txn.go#L655-L745) 関数で処理されます。提供された `error` 自身もしくはその発生源の祖先が `StackTrace()`, `ErrorClass()`, `ErrorAttributes()` メソッドを実装している場合はそれを参照し，無ければ適切な値にフォールバックされます。これに関してはコードを読んでいただくのが一番早いですね…

フォールバックで十分であれば何もしなくてよし，納得いかなければ上記のインタフェースを実装する，あるいはインタフェースを満たしている [`newrelic.Error`](https://github.com/newrelic/go-agent/blob/e9f24662e50b29c3e8eeba5a299780edcda9cc17/v3/newrelic/errors.go#L24-L41) 構造体に詰め替えてもよいでしょう。

```go
// 最小構成ならこれだけで十分
newrelic.FromContext(ctx).NoticeError(err)
```

### [B] HTTP レスポンスステータスコード

New Relic の HTTP インテグレーションを利用している場合， HTTP レスポンスが書き込まれる際のステータスコードがチェックされます。説明用に，[導入ガイド](https://github.com/newrelic/go-agent/blob/e9f24662e50b29c3e8eeba5a299780edcda9cc17/GUIDE.md#transactions)を参照しましょう。

> ```go
> func (h *handler) ServeHTTP(writer http.ResponseWriter, req *http.Request) {
>     txn := h.App.StartTransaction("transactionName")
>     defer txn.End()
>     // This marks the transaction as a web transactions and collects details on
>     // the request attributes
>     txn.SetWebRequestHTTP(req)
>     // This collects details on response code and headers. Use the returned
>     // Writer from here on.
>     writer = txn.SetWebResponse(writer)
>     // ... handler code continues here using the new writer
> }
> ```

ルーターライブラリに合わせて [`nrecho`](https://pkg.go.dev/github.com/newrelic/go-agent/v3/integrations/nrecho-v4) などが使われることがほとんどで，その内部ロジックに焦点が当たることはあまりないですが…上記の `txn.SetWebResponse(writer)` の呼び出しが重要です。これにより，レスポンスステータスコードに基づいてエラーが通知されるようになります。

:::message alert
但し，更に[エラー報告部分の実装](https://github.com/newrelic/go-agent/blob/e9f24662e50b29c3e8eeba5a299780edcda9cc17/v3/newrelic/internal_txn.go#L396-L402)を見ると分かる通り，ここから拾われるエラーは **HTTP ステータスコードの分類しかしてくれません**。もしエラーを詳細に設計しているアプリケーションでの実用性を重視するなら，後述する `ErrorCollector` の設定で全ての HTTP ステータスコードを無視するように設定した上で，自前で用意した HTTP エラーハンドラの中で [A] の明示的なアプローチを取るほうがよいかもしれません。

:::details 例: echo 用のエラーハンドラ
```go
// 事前に newrelic.NewApplication(...) の引数で以下を設定しておく
func(config *newrelic.Config) {
    config.ErrorCollector.IgnoreStatusCodes = []int{
        http.StatusBadRequest,
        http.StatusUnauthorized,
        http.StatusForbidden,
        http.StatusNotFound,
        http.StatusConflict,
        http.StatusGone,
        http.StatusUnprocessableEntity,
        http.StatusInternalServerError,
        http.StatusGatewayTimeout,
    }
}
```

```go
func AppErrorHandler(e *echo.Echo) echo.HTTPErrorHandler {
    defaultHTTPErrorHandler := e.DefaultHTTPErrorHandler

    return func(err error, c echo.Context) {
        var echoError echo.HTTPError

        // echo.HTTPError 型の場合はデフォルトハンドラに委譲
        if errors.As(err, &echoError) {
            defaultHTTPErrorHandler(err, c)

            return
        }

        if c.Response().Committed {
            return
        }

        ctx := c.Request().Context()

        // エラーから HTTP ステータスコードを判定する独自ロジック
        // 例: errors.As() でカスタムエラー型をチェックし，適切なステータスコードを返す
        status := 独自のHTTPステータスコード判定ロジック(err)

        if status >= http.StatusInternalServerError {
            newrelic.FromContext(ctx).NoticeError(err)
            echoError = echo.NewHTTPError(status, "Internal Server Error")
        } else {
            newrelic.FromContext(ctx).NoticeExpectedError(err)
            echoError = echo.NewHTTPError(status, err.Error())
        }

        _ = c.JSON(status, echoError)
    }
}

// e は *echo.Echo 型のインスタンス
e.HTTPErrorHandler = AppErrorHandler(e)
```
:::

### [C] Panic の捕捉

:::message
この機能は [`newrelic.NewApplication()`](https://pkg.go.dev/github.com/newrelic/go-agent#NewApplication) の引数で以下を設定している場合のみ有効です。

```go
func(config *newrelic.Config) {
    config.ErrorCollector.RecordPanics = true
}
```
:::

トランザクションの終端処理として `defer txn.End()` の形で記載していた場合， Panic が発生した場合に `recover()` で捕捉され，エラーとして通知されます。

実際にはミドルウェアレイヤーで `recover()` 処理が入り，それが 500 Internal Server Error に変換されるような実装になっていることがほとんどでしょう。そのためあまり日の目を見ることはないかもしれませんが，保険的にはつけておいて損はない機能だと思います。

# New Relic 導入: API 基本編

さて前置きにかなり時間を使いましたが，ここからはより具体的に導入の話をしていきます。業務で実際に遭遇したハマりポイントは重点的に解説します。それ以外の部分についてもあっさり記載しますが，詳細は公式ドキュメントや別の記事も参照してください。

:::message
理想的には [`newrelic`](https://pkg.go.dev/github.com/newrelic/go-agent) パッケージにべっとり依存した計装コードが書かれるよりは，自前で用意した `internal/telemetry` のようなパッケージで具象を隠蔽することが望ましいです。とはいえこの記事では説明を簡単にするために，直接 [`newrelic`](https://pkg.go.dev/github.com/newrelic/go-agent) パッケージを使うことにします。
:::

## 1. アプリケーション初期化

重要な設定項目についてコメントします。ドキュメントがあまり充実していない部分もあるので，正確な動作を知りたかったらソースコードを読め，みたいな哲学を感じましたね…

既に上の例で `BootstrapNewRelic` 関数のスニペットを示していますが， [`newrelic.NewApplication()`](https://pkg.go.dev/github.com/newrelic/go-agent#NewApplication) の呼び出し部分を引数に注目して掲載します。細かくカスタイマイズしたい要求がない限りは，以下の設定項目だけで十分なはずです。

```go
var err error

// アプリケーション起動時に 1 回だけ実行
App, err = newrelic.NewApplication(
    // ライセンスキー設定
    // 0文字であるか，有効な文字数 (40文字) のキーである必要がある
    newrelic.ConfigLicense("XXXXXXXX...XXXXXXXX"),

    // アプリケーション名
    // 1文字以上の名前を設定する必要がある
    newrelic.ConfigAppName("my-example-app-" + os.Getenv("MY_EXAMPLE_APP_ENV")),

    // Log Forwarding 機能を有効化するか
    // 標準出力からのログ転送が構築しづらいローカル環境で有用
    // デフォルトで true だが環境に応じて明示的に設定を推奨
    newrelic.ConfigAppLogForwardingEnabled(os.Getenv("MY_EXAMPLE_APP_ENV") == "local"),

    // Logs in Context 用の標準出力装飾機能を有効化するか
    // デフォルトで false であるため，もし Log Forwarding 機能が無効である場合は true に設定する必要がある
    // 既に説明した通り， JSONL 形式の出力も末尾への NR-LINKING データ追加で破壊されるので注意
    newrelic.ConfigAppLogDecoratingEnabled(true),

    // トランザクションの defer txn.End() で panic を捕捉し，エラーイベントを APM に報告するかを設定
    // 但し，ここに来るまでの HTTP Middleware レイヤーで recover されていることが多いため，効果が薄い場合がある
    // 保険的な意味合いで有効化しておくのが無難
    func(config *newrelic.Config) {
        config.ErrorCollector.RecordPanics = true
    },

    // New Relic エージェントの内部動作をログ出力するためのロガーを設定
    // デフォルトでは無効，以下のいずれかで設定
    //
    // - newrelic.ConfigDebugLogger(os.Stdout),
    // - newrelic.ConfigInfoLogger(os.Stdout),
    // - newrelic.ConfigLogger(自作ロガー作成(os.Stdout)),
    //
    // 但し，内部ログがアプリケーションのログと混ざって煩雑になるのを回避したければ，
    // 自作ロガーを用意して共通のプレフィクスを付ける，もしくはカスタム属性を付与しておきたいところ
    // newrelic.ConfigDebugLogger(os.Stdout),
)
```

:::message alert
[`newrelic.Application`](https://pkg.go.dev/github.com/newrelic/go-agent#Application) 作成後， API アプリケーションでは待機処理を **入れない** ことが推奨されます。

- エージェントが New Relic に接続するまでに時間がかかる場合があるが，**立ち上がりを待機しているとその間リクエストを受けられない**
- エージェントが接続に失敗しても，アプリケーションの動作自体は継続できる
  - New Relic Go Agent の公開メソッドは全てレシーバの `nil` チェックを行っているため， **`App` が `nil` であっても安全に呼び出せる**
  - 最近は Cloudflare の障害も日常茶飯事だし，いつ New Relic 自体が障害に陥るか分からない…
- エージェントは，バックグラウンドで指数バックオフしながら，成功するまで無制限に再接続を試みる

こういった背景を踏まえると， API では [`App.WaitForConnection(timeout)`](https://pkg.go.dev/github.com/newrelic/go-agent#Application.WaitForConnection) を呼び出さないほうがよいでしょう。 Web 上で目にするサンプルコードの多くはこの処理が入っていますが，この部分は真似しないようにしてください。せめて待機するにしても時間は極力短く，更にエラーが発生してもそれは警告で済ませましょう。（バッチのコード例を参考にしてください）
:::

## 2. HTTP ミドルウェアによる計装

以下に例として [`zerolog`](https://pkg.go.dev/github.com/rs/zerolog) を Logs in Context も兼ねたロガーとして使いつつ， [`lecho`](https://pkg.go.dev/github.com/ziflex/lecho) をアダプターとして使いながら [`echo`](https://pkg.go.dev/github.com/labstack/echo) のミドルウェアとして設定する場合の例を示します。

```go
func NewEcho(ctx context.Context) *echo.Echo {
    e := echo.New()
    e.HideBanner = true
    e.Logger = lecho.From(*log.Ctx(ctx)) // コンテキストロガーを取ってはいるが，この時点ではまだトランザクション情報は含まれていない共通ロガー

    recoverer := func() echo.MiddlewareFunc {
        cfg := middleware.DefaultRecoverConfig

        // 標準のロギング対応ではコンテキストロガーにトランザクション情報が含まれない。
        // Logs in Context を実現するために log.Ctx(c.Request().Context()) を使うようにカスタマイズする
        cfg.LogErrorFunc = func(c echo.Context, err error, stack []byte) error {
            log.Ctx(c.Request().Context()).Error().Err(err).Msgf("[PANIC RECOVER] %v %s\n", err, stack)

            // 上記のログ出力で stack は含められているが，後続のトレースの NoticeError 処理でもスタックトレースを取得してほしい。
            // そのため，スタックトレース情報を付与した新しいエラーを返す
            // （errors.WithStackDepth は cockroachdb/errors パッケージの関数を想定）
            return errors.WithStackDepth(err, 2)
        }

        // Panic を捕捉した上で，それをエラーとして上位のミドルウェアに伝播してもらう
        // false でも設定した AppErrorHandler が呼ばれるが，ここで握りつぶしてしまうと
        // Recover を多段構成にした場合に後続のミドルウェアが実行されなくなるので，一貫性のために true にする
        cfg.DisableErrorHandler = true

        // 他の Goroutine のスタックトレース情報は冗長なので省略する
        cfg.DisableStackAll = true

        return middleware.RecoverWithConfig(cfg)
    }

    e.Use(
        recoverer(), // ミドルウェア内の Panic を捕捉し， error として AppErrorHandler に伝播
        nrecho.Middleware(App), // New Relic トランザクション計装
        lecho.Middleware(lecho.Config{ // リクエスト情報の自動ロギング
            Logger:      e.Logger,
            HandleError: true,

            // New Relic Logs in Context 対応
            Enricher:    func (c echo.Context, logger zerolog.Context) zerolog.Context {
                ctx := c.Request().Context()
                ctx = logger.Logger().WithContext(ctx)
                ctx = WithContextEnrichment(ctx) // ← この記事の中で既に提示した関数を参照
                return log.Ctx(ctx).With()
            },
        }),
        recoverer(), // HTTP ハンドラ内の Panic を捕捉し， error として上位に伝播
    )

    // ルーティング設定など...

    return e
}
```

:::details （参考）lecho 無しでハンドラー内の Logs in Context だけに着目する場合の簡易的なミドルウェア
リクエスト情報自体の自動的なロギングは有効になりませんが，ハンドラー内で [`log.Ctx(ctx)`](https://pkg.go.dev/github.com/rs/zerolog/log#Ctx) を実行したときに Context Enrichment が効いていればいい場合は，以下のようなミドルウェアを用意すれば十分です。

```go
func ContextEnrichmentMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func (c echo.Context) error {
        ctx := c.Request().Context()
        ctx = WithContextEnrichment(ctx) // ← この記事の中で既に提示した関数を参照
        return next(c.WithContext(ctx))
    }
}
```
:::

## 3. 全関数計装

既に紹介した [`nrseg`](https://pkg.go.dev/github.com/budougumi0617/nrseg) をご利用ください。コンテキストにトランザクションを埋め込んでいれば，このライブラリが対応してくれる内容で十分価値は発揮できるはずです。

https://github.com/budougumi0617/nrseg

> 💬 *べっとり [`newrelic`](https://pkg.go.dev/github.com/newrelic/go-agent) パッケージ依存のコードがばら撒かれるのが嫌だって？その場合は `internal/telemetry` パッケージを自前で用意し，それをターゲットにした自作版 [`nrseg`](https://pkg.go.dev/github.com/budougumi0617/nrseg) を作ってください。私は作りましたので（2回目）*

> 💬 [`context.Context`](https://pkg.go.dev/context#Context) [`*http.Request`](https://pkg.go.dev/http#Request) だけじゃなくて [`echo.Context`](https://pkg.go.dev/github.com/labstack/echo#Context) も対応してほしい？ [Issue](https://github.com/budougumi0617/nrseg/issues/17) は上がってるのでコントリビューションのチャンスですよ！私はそこまでの時間は取れませんでしたが，自作版ではこれらすべての型に対応していました。

----

後続の章でエッジケースについてのトラブルシューティングを追記しますが，多くの Web API アプリケーションはこれで対応として十分でしょう。

# New Relic 導入: バッチ編

API アプリケーションとは言えども，バッチ処理は併用されることは多々あります。バッチアプリケーションに New Relic Go Agent を導入する場合， HTTP インテグレーションが利用できないため，トランザクションの開始と終了を手動で行う必要があります。以下に， [`urfave/cli`](https://pkg.go.dev/github.com/urfave/cli) での利用例を示します。

:::message
Web 基本編で，既にグローバル変数 `App` は設定済みだと仮定します。
:::

:::message alert
[`newrelic.Application`](https://pkg.go.dev/github.com/newrelic/go-agent#Application) 作成後，バッチアプリケーションでは待機処理を入れるところまではいいですが， **失敗した場合にもアプリケーションを継続できる** ようなコードにしてください。

バッチアプリケーションは API アプリケーションと異なりライフサイクルが短く， New Relic 接続完了を待たないと重要な処理を見逃してしまうかもしれません。しかし New Relic との接続でエラーが発生した場合に本来の仕事をせずクラッシュしてしまっては大変なことになるので，警告としてログに記録した上で続行するようにしましょう。

```go
if err := App.WaitForConnection(timeout); err != nil {
    log.Ctx(ctx).Warn().Err(err).
        Float64("timeout", timeout.Seconds()).
        Msgf("%.1f 秒待機しましたが New Relic に接続できませんでした。処理は続行されます。", timeout.Seconds())
}
```
:::

## 1. ネストしたコマンドの名称を統合するためのコンテキストを整備

[`urfave/cli`](https://pkg.go.dev/github.com/urfave/cli) ではネストしたコマンドを定義できるため，例えば以下のようなコマンド構成が可能です。

```bash
my-example-app user create
my-example-app user delete
my-example-app user list
```

:::details 上記を構成するコード例
```go
app := &cli.App{
    Name:  "my-example-app",
    Usage: "My Example Application CLI",
    Commands: []*cli.Command{
        {
            Name:  "user",
            Usage: "ユーザー管理",
            Subcommands: []*cli.Command{
                {
                    Name:  "create",
                    Usage: "ユーザーを作成",
                    Action: func(c *cli.Context) error {
                        // 実装...
                        return nil
                    },
                },
                {
                    Name:  "delete",
                    Usage: "ユーザーを削除",
                    Action: func(c *cli.Context) error {
                        // 実装...
                        return nil
                    },
                },
                {
                    Name:  "list",
                    Usage: "ユーザー一覧を表示",
                    Action: func(c *cli.Context) error {
                        // 実装...
                        return nil
                    },
                },
            },
        },
    },
}

if err := app.Run(os.Args); err != nil {
    log.Fatal(err)
}
```
:::

自動計装をするにあたり，例えば `user create` コマンドが実行された場合に，トランザクション名を `user/create` のようにしたいとします。このためには，ネストしたコマンド名を連結してトランザクション名を生成する必要があります。以下にそのためのユーティリティ関数を示します。

```go
type transactionNameKey struct{}

func TransactionNameFromContext(ctx context.Context) string {
    // 設定がない場合は空文字列とする
    name, _ := ctx.Value(transactionNameKey{}).(string)

    return name
}

func WithTransactionName(ctx context.Context, name string) context.Context {
    return context.WithValue(ctx, transactionNameKey{}, name)
}

func WithAppendedTransactionName(ctx context.Context, name string) context.Context {
    parent := TransactionNameFromContext(ctx)

    if parent == "" {
        return WithTransactionName(ctx, name)
    }

    // スラッシュで連結
    return WithTransactionName(ctx, parent+"/"+name)
}
```

ここで定義した関数は次の自動計装ロジックで使用します。

## 2. コマンドハンドラの計装

先程の [`*cli.App`](https://pkg.go.dev/github.com/urfave/cli#App) の定義が `app` 変数に格納されているとします。 [`Before`](https://pkg.go.dev/github.com/urfave/cli#Command.Before) ハンドラを利用してトランザクション名を事前に確定させておき， [`Action`](https://pkg.go.dev/github.com/urfave/cli#Command.Action) ハンドラをラップしてトランザクションを開始・終了させるようにします。以下にそのためのユーティリティ関数群を示します。

:::message
↓書いていてややこしいと思ったのですがすいません…

- `app`: [`*cli.App`](https://pkg.go.dev/github.com/urfave/cli#App)
- `App`: [`*newrelic.Application`](https://pkg.go.dev/github.com/newrelic/go-agent#Application)

実際はパッケージ切ったりして，名前空間は分かれることにはなると思います。
:::

```go
// cli.Command の Before ハンドラを連結するユーティリティ関数
func chainHandlers(handlers ...func(*cli.Context) error) func(*cli.Context) error {
    return func(ctx *cli.Context) error {
        for _, h := range handlers {
            if err := h(ctx); err != nil {
                return err
            }
        }

        return nil
    }
}

// 親コマンド名に連結して現在のコマンド名を設定
func assignNewRelicTransactionName(ctx *cli.Context) error {
    ctx.Context = WithAppendedTransactionName(ctx.Context, ctx.Command.Name)

    return nil
}

// 再帰的にトレーシング機能をインストールする
func installNewRelicFeaturesRecursive(cmd *cli.Command) {
    if cmd.Before != nil {
        cmd.Before = chainHandlers(assignNewRelicTransactionName, cmd.Before)
    } else {
        cmd.Before = assignNewRelicTransactionName
    }

    if cmd.Action != nil {
        cmd.Action = wrapActionHandler(cmd.Action)
    }

    for _, subCmd := range cmd.Subcommands {
        installNewRelicFeaturesRecursive(subCmd)
    }
}

// Action ハンドラをラップしてトランザクションを開始・終了させる
func wrapActionHandler(action cli.ActionFunc) cli.ActionFunc {
    return func(ctx *cli.Context) error {
        // トランザクション開始・自動終了
        txn := App.StartTransaction(TransactionNameFromContext(ctx.Context))
        defer txn.End()

        // コンテキストにトランザクションを埋め込む
        ctx.Context = newrelic.NewContext(ctx.Context, txn)

        // Logs in Context を有効にする
        ctx.Context = WithContextEnrichment(ctx.Context) // ← この記事の中で既に提示した関数を参照

        // 元の Action ハンドラを呼び出す
        if err := action(ctx); err != nil {
            // Logs in Context 対応のもとに New Relic にエラーを報告
            newrelic.FromContext(ctx.Context).NoticeError(err)

            // 終了ステータスを非ゼロにするためにエラーを返す
            return err
        }

        return nil
    }
}
```

これらの関数を呼び出すことで，すべてのコマンドに対してトレーシング機能がインストールされます。

```go
for _, cmd := range app.Commands {
    installNewRelicFeaturesRecursive(cmd)
}
```

これでバッチの全コマンドが，ネストしたコマンドを考慮しながらトランザクション名を設定しつつ， New Relic Go Agent によって自動計装されるようになります。

## 3. 全関数計装

また API 同様に [`nrseg`](https://pkg.go.dev/github.com/budougumi0617/nrseg) が使えると思います。

> 💬 [`context.Context`](https://pkg.go.dev/context#Context) [`*http.Request`](https://pkg.go.dev/http#Request) だけじゃなくて [`*cli.Context`](https://pkg.go.dev/github.com/urfave/cli#Context) も対応してほしい？自作版で対応してください。私は作りましたので（3回目）

----

バッチも基本的にはこれで完結…といきたいところですが，私の前には地獄が待っていました。

# 発展編: トラブルシューティング集

## New Relic Go Agent の内部ロガーがエラーを大量発生

既に `newrelic.ConfigDebugLogger`, `newrelic.ConfigInfoLogger` および `newrelic.ConfigLogger` についてはコメント内で軽く触れていますが，私のプロジェクトでは New Relic の内部ロガーも通常のログ出力に混ぜて使用していました。アプリケーションのログ， New Relic Go Agent の内部ログが両方とも同じ標準出力に流れ，これらが New Relic に転送されていました。

この記事で触れている全関数計装を実装し終わったと思って一段落していた頃，検証環境で New Relic の内部ロガーがとんでもない数のエラーを発生させていることに気が付きました。

```json
{
    "message": "unable to end segment",
    "reason": "improper segment use: segments must be ended in \"last started first ended\" order: use https://godoc.org/github.com/newrelic/go-agent/v3/newrelic#Transaction.NewGoroutine to use the transaction in multiple goroutines"
}
```

```go
// ↓ この .End() で大量のエラーが発生していた
defer newrelic.FromContext(ctx).StartSegment("Segment Name").End()
```

https://qiita.com/masahiro_takeda/items/98934279aaa4c7774453

結論から言うと…上の方と全く同じ道を通っていたことになったのですが，問題分析に時間がかかっていたのと今後の安全のため，以下の 2 つの対応を検討しました。

- New Relic Go Agent 内部ロガーを無効化する
- New Relic Go Agent 内部ロガー上の同一エラーをスロットリングする

無効化してしまえば手っ取り早かったのですが，エージェント自体のヘルスチェックもあったほうが望ましいため，後者の同一ログスロットリング方式を採用しました。

:::details 指数バックオフスロットリングを適用した New Relic Go Agent 内部ロガーの例
- New Relic Go Agent の内部ログとして流れてくる `message` の種類数は限られていますが，念の為 LRU キャッシュで無制限に増えてもメモリを浪費しないように工夫しています。
- 同一の `message` ごとに `throttlingState` を持ち，その中でバースト数とバックオフ時間を管理しています。

```go
import (
    "sync"
    "time"

    lru "github.com/hashicorp/golang-lru/v2"
    "github.com/newrelic/go-agent/v3/newrelic"
    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

const (
    logThrottlingLRUSize = 50
    logThrottlingBurstCount = 10
    logThrottlingInitialDuration = 1 * time.Second
    logThrottlingMaxDuration = 30 * time.Second
    logThrottlingBurstResetDuration = 60 * time.Second
)

type throttlingState struct {
    lastSeenAt      time.Time
    backoffDuration time.Duration
    count           int
    mu              *sync.Mutex
}

type internalLogger struct {
    debug bool
    lru   *lru.Cache[string, *throttlingState]
}

func newInternalLogger(debug bool) *internalLogger {
    cache, _ := lru.New[string, *throttlingState](logThrottlingLRUSize)

    return &internalLogger{
        debug: debug,
        lru:   cache,
    }
}

func (l *internalLogger) log(level zerolog.Level, msg string, context map[string]any) {
    e := log.
        WithLevel(level).
        CallerSkipFrame(2)

    for k, v := range context {
        e = e.Any(k, v)
    }

    e.Msg(msg)
}

func (l *internalLogger) Error(msg string, context map[string]any) {
    if !l.shouldLog(msg) {
        return
    }

    l.log(zerolog.ErrorLevel, msg, context)
}

func (l *internalLogger) Warn(msg string, context map[string]any) {
    if !l.shouldLog(msg) {
        return
    }

    l.log(zerolog.WarnLevel, msg, context)
}

func (l *internalLogger) Info(msg string, context map[string]any) {
    l.log(zerolog.InfoLevel, msg, context)
}

func (l *internalLogger) Debug(msg string, context map[string]any) {
    if !l.debug {
        return
    }

    l.log(zerolog.DebugLevel, msg, context)
}

func (l *internalLogger) DebugEnabled() bool {
    return l.debug
}

func (l *internalLogger) shouldLog(msg string) bool {
    now := internalLoggerNowFunc()

    state, ok := l.lru.Get(msg)
    if !ok {
        state = &throttlingState{
            lastSeenAt:      now,
            backoffDuration: logThrottlingInitialDuration,
            count:           1,
            mu:              new(sync.Mutex),
        }
        l.lru.Add(msg, state)

        return true
    }

    state.mu.Lock()
    defer state.mu.Unlock()

    if now.Sub(state.lastSeenAt) > logThrottlingBurstResetDuration {
        state.count = 0
    }

    if state.count < logThrottlingBurstCount {
        state.lastSeenAt = now
        state.count++

        return true
    }

    if now.Sub(state.lastSeenAt) >= state.backoffDuration {
        state.lastSeenAt = now

        if state.backoffDuration < logThrottlingMaxDuration {
            state.backoffDuration *= 2
            if state.backoffDuration > logThrottlingMaxDuration {
                state.backoffDuration = logThrottlingMaxDuration
            }
        }

        return true
    }

    return false
}
```
:::

これで今後同種のエラーが発生しても， New Relic のログクォータを New Relic Go Agent の内部ログに食い潰されてしまうことは回避できます。さて，以下ではこの種の問題に対する本質的な解決策を記載します。

## Goroutine 対応不備 ①: Goroutine による並行処理でセグメント計測エラー

New Relic Go Agent のトランザクションやセグメントは Goroutine セーフではありません。 Goroutine 内でセグメントを開始・終了しようとすると，以下のようなエラーが発生します。

> ```json
> {
>     "message": "unable to end segment",
>     "reason": "improper segment use: segments must be ended in \"last started first ended\" order: use https://godoc.org/github.com/newrelic/go-agent/v3/newrelic#Transaction.NewGoroutine to use the transaction in multiple goroutines"
> }
> ```

最初のほうで以下のような図を記載しました。

> ```
> newrelic.Application (アプリケーション全体)
> └── newrelic.Transaction (1 つのリクエスト処理)
>     └── newrelic.Segment (関数呼び出し)
> ```

では，ネストした関数コールの各階層で連続的に発生する，複数のセグメントが計測されるときはどうなるのでしょうか？

```
newrelic.Application
└── A. newrelic.Transaction (Goroutine A)
    ├── a. newrelic.Segment (Goroutine A: 1回目)
    ├── b. newrelic.Segment (Goroutine A: 2回目)
    └── c. newrelic.Segment (Goroutine A: 3回目)
```

|               Goroutine A |
|--------------------------:|
| `A := StartTransaction()` |
|                         ⋮ |
|   `a := A.StartSegment()` |
|   `b := A.StartSegment()` |
|   `c := A.StartSegment()` |
|              `c.End()` 🆗 |
|              `b.End()` 🆗 |
|              `a.End()` 🆗 |
|                         ⋮ |
|              `A.End()` 🆗 |

セグメントは上記のように **LIFO（後入れ先出し）** の順序で終了されることによって，入れ子関係が正しく計測されます。 **セグメントはトランザクションの子要素として横並びになるだけで，セグメント同士が親子関係になるわけではない** ため，このような制約があるのです。

では，もしここで Goroutine 間でトランザクションを共有してしまったらどうなるでしょうか？

```
newrelic.Application
└── A. newrelic.Transaction (Goroutine A/B)
    ├── a. newrelic.Segment (Goroutine A: 1回目)
    ├── b. newrelic.Segment (Goroutine A: 2回目)
    ├── c. newrelic.Segment (Goroutine B: 1回目)
    ├── d. newrelic.Segment (Goroutine A: 3回目)
    └── e. newrelic.Segment (Goroutine B: 2回目)
```

|               Goroutine A | Goroutine B             |
|--------------------------:|:------------------------|
| `A := StartTransaction()` | ↴                       |
|                         ⋮ | ⋮                       |
|   `a := A.StartSegment()` | ⋮                       |
|   `b := A.StartSegment()` | ⋮                       |
|                         ⋮ | `c := A.StartSegment()` |
|   `d := A.StartSegment()` | ⋮                       |
|                         ⋮ | `e := A.StartSegment()` |
|                         ⋮ | `e.End()` 🆗            |
|              `d.End()` 🆗 | ⋮                       |
|              `b.End()` 💥 | ⋮                       |

Goroutine 間は並行処理されるため， LIFO の順序でセグメントが終了するとは限りません。これがトランザクションが Goroutine セーフではない直接的な理由です。 [`sync.Mutex`](https://pkg.go.dev/sync#Mutex) を使っているかどうかという話ではなく，**セグメントの開始・終了の順序が Goroutine 間で入り乱れてしまうため，正しい順序で終了できなくなる** のです。

この問題を解決するために， New Relic Go Agent では [`(*newrelic.Transaction).NewGoroutine()`](https://pkg.go.dev/github.com/newrelic/go-agent#Transaction.NewGoroutine) というメソッドが提供されています。

```go
defer newrelic.FromContext(ctx).StartSegment("Root Transaction Segment").End()

wg := new(sync.WaitGroup)
wg.Go(func() {
    // この Goroutine 用に Transaction を派生
    ctx := newrelic.NewContext(ctx, newrelic.FromContext(ctx).NewGoroutine())

    defer newrelic.FromContext(ctx).StartSegment("Derived Transaction Segment").End()

    // Do something in Goroutine...
})
wg.Wait()
```

:::details sync.WaitGroup の古い使い方をする場合
この場合は，外側で作っておいたコンテキストを引数で渡すほうが綺麗かもしれません。とはいえ，このためだけにレガシーな呼び出し方をする必要は無さそうですが…

```go
defer newrelic.FromContext(ctx).StartSegment("Root Transaction Segment").End()

wg := new(sync.WaitGroup)
wg.Add(1)

go func (ctx context.Context) {
    defer wg.Done() // 要注意: セグメント終了記録処理よりも後に `.Done()` が走るようにする！
    defer newrelic.FromContext(ctx).StartSegment("Derived Transaction Segment").End()

    // Do something in Goroutine...
}(
    // この Goroutine 用に Transaction を派生したコンテキストを渡す
    newrelic.NewContext(ctx, newrelic.FromContext(ctx).NewGoroutine()),
)

wg.Wait()
```
:::

これを使うと，以下のように Goroutine ごとに派生トランザクションを持つことができ，セグメントの開始・終了順序が混在することを防げます。

```
newrelic.Application
└── A. newrelic.Transaction (Goroutine A) ────────── B. newrelic.Transaction (Goroutine B)
    ├── a. newrelic.Segment (Goroutine A: 1回目)      ├── c. newrelic.Segment (Goroutine B: 1回目)
    ├── b. newrelic.Segment (Goroutine A: 2回目)      └── e. newrelic.Segment (Goroutine B: 2回目)
    └── d. newrelic.Segment (Goroutine A: 3回目)
```

|               Goroutine A | Goroutine B             |
|--------------------------:|:------------------------|
| `A := StartTransaction()` | ↴                       |
|                         ⋮ | `B := A.NewGoroutine()` |
|                         ⋮ | ⋮                       |
|   `a := A.StartSegment()` | ⋮                       |
|   `b := A.StartSegment()` | ⋮                       |
|                         ⋮ | `c := B.StartSegment()` |
|   `d := A.StartSegment()` | ⋮                       |
|                         ⋮ | `e := B.StartSegment()` |
|                         ⋮ | `e.End()` 🆗            |
|              `d.End()` 🆗 | ⋮                       |
|            `b.End()` 🆗✨️ | ⋮                       |
|                         ⋮ | `c.End()` 🆗            |
|              `a.End()` 🆗 | ⋮                       |
|                         ⋮ | ⋮                       |
|                         ⋮ | `B.End()` 🆗            |
|              `A.End()` 🆗 |                         |

:::details コラム: New Relic 以外はどうなのよ？
Datadog および OpenTelemetry は Goroutine セーフです。スパン開始時に `ctx` 変数を置き換えるお作法になっているため，実質的に New Relic でいうところの [`(*newrelic.Transaction).NewGoroutine()`](https://pkg.go.dev/github.com/newrelic/go-agent#Transaction.NewGoroutine) を呼び出しているのと同じ効果が得られています。

**Datadog の場合:**

```go
span, ctx := tracer.StartSpanFromContext(ctx, "operation.name")
defer span.Finish()
```

**OpenTelemetry の場合:**

```go
ctx, span := otel.Tracer("my-example-app").Start(ctx, "operation.name")
defer span.End()
```

どちらも **スパン開始時に必ず新しいコンテキストが返される** 設計になっているため，開発者が意識せずとも自然と Goroutine セーフになります。一方 New Relic は既存のトランザクションをコンテキストから取り出して使い回す設計なので， [`(*newrelic.Transaction).NewGoroutine()`](https://pkg.go.dev/github.com/newrelic/go-agent#Transaction.NewGoroutine) の呼び出しを明示的に行う必要があります。

> 💬オーバーヘッドとか微々たるものだろうし，こっちのほうがいいのでは…？
> 💬でも全関数計装でコンテキスト毎回ラップしてたら流石にやばいか…？
> 💬`defer` ステートメントが New Relic だけ 1 行で書けるのは救いかな…？
↓
> 💬**[Context-induced performance bottleneck in Go - Gab's Notes](https://gabnotes.org/posts/context-induced-performance-bottleneck-in-go) とか読むと，やっぱり全関数計装前提なら New Relic が正しい気がしてくる…！**
:::

## Goroutine 対応不備 ②: Goroutine による遅延処理でセグメント計測エラー

並行処理を Goroutine で行い，後に [`sync.WaitGroup`](https://pkg.go.dev/sync#WaitGroup) などで待機する場合は問題なかったのですが， 「重たい処理はレスポンスを返した後に遅延させる」など， **分岐した Goroutine のほうがリクエストを処理する Goroutine よりも長生きする** 場合はどうでしょうか？

:::message alert
**[`(*newrelic.Transaction).NewGoroutine()`](https://pkg.go.dev/github.com/newrelic/go-agent#Transaction.NewGoroutine) で派生したトランザクションは，親トランザクションが終了した時点で無効** になってしまいます。

|               Goroutine A | Goroutine B                 |
|--------------------------:|:----------------------------|
| `A := StartTransaction()` | ↴                           |
|                         ⋮ | `B := A.NewGoroutine()`     |
|                 `A.End()` | ↲                           |
|                           | &nbsp;                      |
|                           | `x := B.StartSegment()` 🆗❓ |
|                           | `x.End()` 💥                |

上記の `x.End()` のタイミングでエラーが発生します。
:::

これは特大トラップですね。対応が終わって安心仕切っていたところで，また New Relic Go Agent の内部ロガーにエラー爆撃を喰らいました…

```json
{
    "message": "unable to end segment",
    "reason": "transaction has already ended"
}
```

ではどうやって対応するのがいいでしょうか？正解はこうです。

```
newrelic.Application
├─── A. newrelic.Transaction (Goroutine A)
│    ├── a. newrelic.Segment (Goroutine A: 1回目)
│    ├── b. newrelic.Segment (Goroutine A: 2回目)
│    └── c. newrelic.Segment (Goroutine A: 3回目)
└─── B. newrelic.Transaction (Goroutine B)
     ├── d. newrelic.Segment (Goroutine B: 1回目)
     └── e. newrelic.Segment (Goroutine B: 2回目)
```

|               Goroutine A | Goroutine B                  |
|--------------------------:|:-----------------------------|
| `A := StartTransaction()` |
|                         ⋮ |                              |
|   `a := A.StartSegment()` |                              |
|   `b := A.StartSegment()` |                              |
|   `c := A.StartSegment()` |                              |
|              `c.End()` 🆗 |                              |
|             `b.End()` 🆗️ |                              |
|                         ⋮ | `B := StartTransaction()`    |
|              `a.End()` 🆗 | ⋮                            |
|                         ⋮ | ⋮                            |
|                         ⋮ | `d := B.StartSegment()`      |
|               `A.End()` ️ | ⋮                            |
|                           | ⋮                            |
|                           | `e := B.StartSegment()` 🆗✨️ |
|                           | `e.End()` 🆗✨️               |
|                           | `d.End()` 🆗✨️               |
|                           | ⋮                            |
|                           | `B.End()` 🆗                 |

元の設計の問題点は派生トランザクションを作ってしまっていたことでした。つまり派生ではなく， **全く別のトランザクションとして作れば問題は解決するのです**。

```go
defer newrelic.FromContext(ctx).StartSegment("Root Transaction Segment").End()

go func (ctx context.Context) {
    txn := App.StartTransaction("Background Job Transaction")
    defer txn.End()

    ctx = newrelic.NewContext(ctx, txn)

    defer newrelic.FromContext(ctx).StartSegment("Background Job Segment").End()

    // Do something in Goroutine...
}(ctx)
```

とはいえ，設計上の都合で APM ダッシュボードで分断されてしまうほど悲しいことはないでしょう。バックグラウンドトランザクションがどの API トランザクションから発生したのか，自然に追跡できるほうが嬉しいですよね。

安心してください， New Relic には **分散トレーシング (Distributed Tracing)** という機能があります。本来は物理的に離れたサービス間でのトレーシングを実現するための機能ですが，これを **同一プロセス上の別のトランザクションを関連付ける** ことにも転用できます。元の用途の都合上， HTTP ヘッダーの知識が溢れる形になってしまいますので，以下のように抽象化したネーミングを使ってみてはどうでしょうか？

```go
// HTTP ヘッダーの知識を隠蔽するためのキャリア構造体
type TracePropagationCarrier struct {
    headers http.Header
}

// 伝播元で使用する関数
func PrepareTracePropagationCarrier(ctx context.Context) *TracePropagationCarrier {
    carrier := &TracePropagationCarrier{
        headers: http.Header{},
    }

    newrelic.FromContext(ctx).InsertDistributedTraceHeaders(carrier.headers)

    return carrier
}

// 伝播先で使用する関数
func ContinueTraceFrom(ctx context.Context, carrier *TracePropagationCarrier) {
    newrelic.FromContext(ctx).AcceptDistributedTraceHeaders(newrelic.TransportOther, carrier.headers)
}
```

```go
defer newrelic.FromContext(ctx).StartSegment("Root Transaction Segment").End()

carrier := PrepareTracePropagationCarrier(ctx) // 伝播情報を準備

go func (ctx context.Context, carrier *TracePropagationCarrier) {
    txn := App.StartTransaction("Background Job Transaction")
    defer txn.End()

    ctx = newrelic.NewContext(ctx, txn)
    ContinueTraceFrom(ctx, carrier) // 伝播情報を受け入れ

    defer newrelic.FromContext(ctx).StartSegment("Background Job Segment").End()

    // Do something in Goroutine...
}(ctx, carrier)
```

ここでは API アプリケーションを想定しましたが，バッチ処理でもこういった遅延処理がある場合は同様に対応できます。汎用性は高いです。

## Linter が欲しい！

さて，ここまで相当苦労を積んできましたが，結局正しく計装できるかどうかは，エンジニアの注意力に依存している部分があります。

- [`zerolog`](https://pkg.go.dev/github.com/rs/zerolog) の [`log.Ctx(ctx)`](https://pkg.go.dev/github.com/rs/zerolog/log#Ctx) のコンテキスト呼び出し漏れで Logs in Context 非対応のロギング処理が散在している
- 並行処理での [`(*newrelic.Transaction).NewGoroutine()`](https://pkg.go.dev/github.com/newrelic/go-agent#Transaction.NewGoroutine) 対応漏れ
- 遅延処理での分散トレーシング対応漏れ

全部あるあるです。Linter，欲しいですよね。私も欲しいのでプロジェクト内部用ですが最低限の目的を果たせるものを作りました。これを読んでいるあなたも是非，プロジェクトメンバーのヒューマンエラーをカバーするため，頑張ってください。
（ここの解説を始めると，この記事を読んでくれる人がいなくなるぐらい分量がとんでもないことになるはず…）

一番上の [`zerolog`](https://pkg.go.dev/github.com/rs/zerolog) の件については， [`zerologlint`](https://github.com/ykadowak/zerologlint) という Linter に Feature Request を送りました。ライブラリ品質で仕上げるのはかなり難しいと思いますが，もし興味がある方はコントリビューションしてみてください。

https://github.com/ykadowak/zerologlint/issues/22

# あとがき

全関数計装前提での記事になってしまいましたが，実際には通信境界を計装するだけでも十分仕事してくれると思います。 [AST](https://pkg.go.dev/go/ast) や [DST](https://pkg.go.dev/github.com/dave/dst) を用いたソースコードの解析・自動編集に興味がある方は是非チャレンジしてほしいですが，そんなことをしなくても気軽に簡単なところから始められるので，是非とも New Relic の機能（とくに Logs in Context）を活用してみてください。

また New Relic 自身が OpenTelemetry ファーストを謳っているのもあり，今後 New Relic Go Agent を直接使うことはだんだんと減っていくのではないかと思います。ですが現時点では OpenTelemetry 側が成長過程ということもあるので（~~ちょうど業務で既にそういうコードが書かれていたので仕方なく書きましたが~~），まだまだ New Relic Go Agent の導入ノウハウを共有することには価値があると考え，この記事を書かせていただきました。きっと状況が変わってしまった数年後でも，検索でフラッと出てきて，保守対応を任されている誰かのお役に立てれば本望です。
