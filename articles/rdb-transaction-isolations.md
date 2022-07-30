---
title: "MySQL/Postgres におけるトランザクション分離レベルと発生するアノマリーを整理する"
emoji: "🔒"
type: "tech"
topics: ["database", "transaction", "isolation", "postgresql", "mysql"]
published: false
---

# 読者対象

- ANSI 定義の古典的なトランザクション分離レベルとアノマリーは概ね理解している
- MySQL/Postgres では理論的な部分がどうなっているのかを知りたい

# 理論面の前提知識

まず ANSI 定義の古典的な定義を聞いたことが無い方は，以下のリンクを参照されたい。 ANSI 定義に対応する解説はこれらのサイト以外にもたくさんあるため，自分にとって読みやすいと感じる情報をあたってほしい。（既に熟知されている方は十分）

https://itmanabi.com/transaction-isolation/

https://itsakura.com/sql-isolation

https://qiita.com/song_ss/items/38e514b05e9dabae3bdb

https://qiita.com/hatsu/items/4e699ad50651a6a30407

次点で読んでいただきたいのが， [@kumagi](https://zenn.dev/kumagi) さんの以下の記事。古典的には 4 つの分離レベルと 3 つのアノマリーだけで説明されていたものの，不十分であることが学術的に指摘され，解像度を上げようとする流れが後になって起こってきた。

https://qiita.com/kumagi/items/1dc1a91ec007365ac694

https://qiita.com/kumagi/items/5ef5e404546736ebac49

これらの記事を踏まえた上で，問題提起の発端となった論文に対する以下の解説記事を読むと，理解が深まると思われる。

https://developer.hatenastaff.com/entry/2017/06/21/100000

以下での具体面へのアプローチは，ここまでの知識がインプットされている前提であるとする。

# MySQL/Postgres の実装はどうなっているか？

## 参考文献

論文で提唱されているトランザクション分離レベルを MySQL と Postgres にどう対応させるかについては，以下の記事を参考にした。

http://blog.kimuradb.com/?eid=877571

https://www.kimullaa.com/entry/2020/03/14/134232

更に Postgres の実装詳細については，以下の記事が大きく理解を助けることになった。

http://www.nminoru.jp/~nminoru/postgresql/pg-transaction-mvcc-snapshot.html

https://blog.yux3.net/entry/2021/05/15/102916

これらから得た知識をもとに，表および箇条書きの形式で要点をまとめる。

## 共通事項

#### Anomaly

ANSI 定義より後に新しく登場したものは太字で表現する。ANSI 定義では読み取りの不整合にしか着目されていなかったが，新しい定義では更新競合と直列化異常にも関心が広げられている。

| グルーピング  | 現象                                                        |
|:--------|:----------------------------------------------------------|
| 読み取り不整合 | Dirty Read<br>Fuzzy Read<br>Phantom Read<br>**Read Skew** |
| 更新競合    | **Cursor Lost Update**<br>**Lost Update**                 |
| 直列化異常   | **Write Skew**<br>**Observe Skew**                        |

以下では，それぞれのアノマリーがどういった現象を指しているのかを説明する。

| 現象                                      | 意味                                                                                                                                                    |
|:----------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------|
| Dirty Read                              | 他のトランザクションでコミットされていない変更を参照してしまう                                                                                                                       |
| Fuzzy Read<br>Phantom Read<br>Read Skew | 他のトランザクションでコミットされた変更を参照してしまう                                                                                                                          |
| Cursor Lost Update                      | 他のトランザクションで **Locking Read によって読み取られた後**，<br>コミットされた変更を **上書き** してしまう                                                                                 |
| Lost Update                             | 他のトランザクションでコミットされた変更を **上書き** してしまう                                                                                                                   |
| Write Skew                              | トランザクション $T_1$ が読み取った $x$ を使って $y$ の変更するとき，<br>トランザクション $T_2$ が読み取った $y$ の値を使って $x$ を変更してしまい，<br>**すれ違いざまに相手の変更前の値に依存した更新を行ってしまう**                    |
| Observe Skew                            | 2 つだけであれば直列化可能であったはずのトランザクション $T_1$ $T_2$ の<br>**途中状態**を $T_3$ が観測することによって **循環参照** が発生し，観測者を含めた<br>**$T_1$ $T_2$ $T_3$ の並び順を矛盾なく確定させることができなくなってしまう** |

- MySQL/Postgres とも， **Snapshot Isolation** が採用されているがゆえに Fuzzy Read と Phantom Read はセットで考えることができ，新規・更新・削除の区別を意識する機会は少ない。ここでは簡単のため 1 つにまとめる。
- 2 値の読み取り整合性違反となる Read Skew については，1 つの値を複数回読み取る際の不整合である Fuzzy Read と現象の内容が似ている。これも簡単のために 1 つにまとめる。
- Observe Skew については Read Only Anomaly, Read Only Skew, Batch Processing などさまざまな呼称が存在するが，どれも意味が伝わりにくいので，この記事で Observe Skew という名称を新しく提案する。（論文の中で提案されているものではない）

#### Snapshot Isolation

発行する SQL 文の種類によって，スナップショットと現在のデータ本体 (Current) のどちらを参照するかが異なっている。

| 文法                               | アクション           | 参照先      | ロック       |
|:---------------------------------|:----------------|:---------|:----------|
| `SELECT`                         | Consistent Read | Snapshot | -         |
| `SELECT ... FOR SHARE`           | Locking Read    | Current  | Shared    |
| `SELECT ... FOR UPDATE`          | Locking Read    | Current  | Exclusive |
| `INSERT`<br>`UPDATE`<br>`DELETE` | Write           | Current  | Exclusive |

- **一貫性読み取り (Consistent Read)** では，更新時に参照するデータ本体とは隔離された読み取り専用の **スナップショット** をロックせずに取得する。
- **ロック読み取り (Locking Read)** では，データ本体をロックして取得する。

## MySQL

#### Anomaly

| 現象＼分離レベル                                | READ<br>UNCOMMITTED | READ<br>COMMITTED |  **REPEATABLE READ**<br>**[Default]**   | SERIALIZABLE |
|:----------------------------------------|:-------------------:|:-----------------:|:---------------------------------------:|:------------:|
| Dirty<br>Read                           |          ❌          |         ✅         |                    ✅                    |      ✅       |
| Fuzzy Read<br>Phantom Read<br>Read Skew |          ❌          |         ❌         | 🔺<br>**Broken on**<br>**Locking Read** |      ✅       |
| Cursor Lost Update                      |          ❌          |         ✅         |                    ✅                    |      ✅       |
| Lost Update                             |          ❌          |         ❌         |                    ❌                    |      ✅       |
| Write Skew                              |          ❌          |         ❌         |                    ❌                    |      ✅       |
| Observe Skew                            |          ❌          |         ❌         |                    ❌                    |      ✅       |

#### Locking

| アクション＼分離レベル        | READ UNCOMMITTED<br>READ COMMITTED | **REPEATABLE READ**<br>**[Default]** |      SERIALIZABLE      |
|:-------------------|:----------------------------------:|:------------------------------------:|:----------------------:|
| Consistent Read    |                 -                  |                  -                   | **全て Locking Read 化**  |
| Locking Read/Write |              レコードロック               |        レコードロック<br>**ギャップロック**        | レコードロック<br>**ギャップロック** |

#### Errors

| 原因     | メッセージ例                                                                                    |
|:-------|:------------------------------------------------------------------------------------------|
| デッドロック | ERROR 1213 (40001): Deadlock found when trying to get lock;<br>try restarting transaction |

#### 解説

MySQL は以下のような特徴を持つ。

- ロックとして採用されているものはすべて **悲観ロック** である。
- 分離レベルによらず，トランザクションは常に **後勝ち** で処理される。悲観ロックにより競合したトランザクションは，先行トランザクションがコミットするまで待機させれ，それに続いて後続トランザクションが処理を続行する。

そのほか，トランザクション分離レベルごとに以下のような特徴がある。

- `READ COMMITTED` 以下では，
  - 毎回の操作で最新状態に基づいたスナップショットを取得する。
- `REPEATABLE READ` 以上では，
  - レコードロックでは担保できない現在レコードが存在しない範囲のロックに関しては， **ギャップロック** で解決している。
  - **`SELECT` 実行で取得したスナップショットバージョン** を，自身が変更しない限りはトランザクションの終了まで保持し続ける。同じクエリに対しては同じ結果が保証される。
  - もし **`BEGIN` の代わりに `START TRANSACTION WITH CONSISTENT SNAPSHOT` を使用すると，トランザクション開始と同時にスナップショットを取得する。** 但し，スナップショット取得後の動作に関しては通常の `REPEATABLE READ` との差異は無い。
- `SERIALIZABLE` では， 
  - **Consistent Read はすべて Locking Read に変換される** ため，実質的に存在しない。
  - Lost Update, Write Skew, Observe Skew という直列化異常を全て防ぐことができる。逆に言えば，これらを防ぎたければ Locking Read または `SERIALIZABLE` 分離レベルを使用しなければならない。

:::message alert
**`REPEATABLE READ` 以上でも，Locking Read/Write では `READ COMMITTED` 相当の動作になってしまう仕様となっている。** またそれゆえに，**`REPEATABLE READ` でも Lost Update は発生する。**
- Consistent Read だけをしている限りでは，スナップショットバージョンが固定されているので，読み取り不整合は発生しない。
- Locking Read/Write だけをしている限りでは， **ギャップロック** があるため，読み取り不整合は発生しない。
- **Consistent Read と Locking Read/Write の間に整合性はない。** Consistent Read では現れなかった変更が， Locking Read/Write で現れてしまう場合がある。これは複数回実行した場合も該当する。つまり，以下のような状況があり得る。

```sql
BEGIN;
SELECT COUNT(*) FROM example; -- 100
SELECT COUNT(*) FROM example FOR UPDATE; -- 101
SELECT COUNT(*) FROM example; -- 100
SELECT COUNT(*) FROM example FOR UPDATE; -- 101
```
:::

:::message
ギャップロックがあることによってデッドロックが発生する機会が増加する弊害がある。 **レコードロックの空振りがギャップロックを引き起こすこともある。** またギャップロックの実装上の都合で存在する **ネクストキーロック** は，直感的なロック範囲よりも広い範囲をロックしてしまうことがある。

- [ネクストキーロックとは | ソフトウェア雑記](https://softwarenote.info/p1067/)
:::

https://dev.mysql.com/doc/refman/8.0/ja/innodb-transaction-isolation-levels.html

#### 総評

MySQL は MVCC のための Snapshot Isolation を採用しつつも，基本的な戦略を **「悲観ロック」** **「後勝ちトランザクション」** に寄せているシンプルな RDBMS だと言える。 Locking Read で分離レベルをダウングレードさせるなど，思い切った設計にしている面はあるが，それらを受け入れた上で気をつけて使いたい。

## Postgres

#### Anomaly

|                現象＼分離レベル                 | READ COMMITTED |                  REPEATABLE READ                  |                   SERIALIZABLE                    |
|:---------------------------------------:|:--------------:|:-------------------------------------------------:|:-------------------------------------------------:|
|               Dirty Read                |       ✅        |                         ✅                         |                         ✅                         |
| Fuzzy Read<br>Phantom Read<br>Read Skew |       ❌        |                         ✅                         |                         ✅                         |
|           Cursor Lost Update            |       ✅        |                         ✅                         |                         ✅                         | 
|               Lost Update               |       ❌        | ✅<br>**Concurrent Update**<br>**Error Detection** | ✅<br>**Concurrent Update**<br>**Error Detection** |
|               Write Skew                |       ❌        |                         ❌                         | ✅<br>**R/W Dependencies**<br>**Error Detection**  |
|              Observe Skew               |       ❌        |                         ❌                         | ✅<br>**R/W Dependencies**<br>**Error Detection**  |

#### Locking

| アクション＼分離レベル        | READ COMMITTED | **REPEATABLE READ**<br>**[Default]** |              SERIALIZABLE               |
|:-------------------|:--------------:|:------------------------------------:|:---------------------------------------:|
| Consistent Read    |       -        |                  -                   |             **SIRead ロック**              |
| Locking Read/Write |    レコードロック     |        レコードロック<br>**更新競合検査**         | レコードロック<br>**更新競合検査**<br>**SIRead ロック** |

:::message
**SIRead ロック (Snapshot Isolation Read ロック)** とは，直列化異常検知のための **楽観ロック** を指す。 `SELECT` したデータを SIRead ロックと称して検知のためにマーキングし，後から参照関係をトラバーサルできるようにしておく。また SIRead ロックを取り入れた Snapshot Isolation (SI) は，  **SSI (Serializable Snapshot Isolation)** と呼ばれる。
:::

:::message
厳密には「更新競合検査」はロック処理に分類すべきではないが，ここではまとめて SIRead ロックとまとめて，楽観ロックに準じた動きをさせるものとして扱う。
:::

#### Errors

| 原因     | メッセージ例                                                                                                                                                                                                             |
|:-------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| デッドロック | ERROR:  deadlock detected<br>DETAIL:<br>Process 37542 waits for ShareLock on transaction 2995;<br>blocked by process 37541.<br>Process 37541 waits for ShareLock on transaction 2996;<br>blocked by process 37542. |
| 更新競合   | ERROR:  could not serialize access due to concurrent update                                                                                                                                                        |
| 直列化異常  | ERROR:<br>could not serialize access due to read/write dependencies among transactions<br>DETAIL:<br>Reason code: Canceled on identification as a pivot, during commit attempt.                                    |

#### 解説

Postgres は以下のような特徴を持つ。

- 悲観ロックはレコードロックのみに抑え，それ以外は軽量な楽観ロックで実現するために **更新競合検査** と **SIRead ロック** を導入している。 
- 悲観ロックにより競合したトランザクションは，**後勝ち** で処理される。先行トランザクションがコミットするまで待機させれ，それに続いて後続トランザクションが処理を続行する。
- 楽観ロックにより競合したトランザクションは，**先勝ち** で処理される。先行トランザクションが競合するデータをコミットしている場合，後続トランザクションは中止される。

楽観ロックについては，表面的な動きだけに着目するのであれば， **「トランザクション開始時から全テーブル・全レコードに対して楽観ロックがかけられている」** と考えても差し支えない。そのほか，トランザクション分離レベルごとに以下のような特徴がある。

- `READ COMMITTED` 以下では，
  - 毎回の操作で最新状態に基づいたスナップショットを取得する。
- `REPEATABLE READ` 以上では，
  - **トランザクション開始と同時に** スナップショットを取得し，自身が変更しない限りはトランザクションの終了まで保持し続ける。
  - レコードロックでは担保できない現在レコードが存在しない範囲のロックに関しては， **更新競合検査** で解決している。更新しようとした内容が更新競合検査に引っかかった場合，トランザクションを更新競合エラーとして中止する。
- `SERIALIZABLE` では，
  - レコードロックと更新競合検査でも検知できない直列化異常に関しては， **SIRead ロック** で解決している。更新しようとした内容が SIRead ロックに引っかかった場合，トランザクションを直列化異常エラーとして中止する。
  - Lost Update, Write Skew, Observe Skew という直列化異常を全て防ぐことができる。逆に言えば，これらを防ぎたければ Locking Read または `SERIALIZABLE` 分離レベルを使用しなければならない。

:::message
更新競合検査と SIRead ロックによってエラーが発生する機会が増えるので，キャッチした上でリトライするなど適切にエラーハンドリングする必要がある。
:::

https://www.postgresql.jp/docs/12/transaction-iso.html

