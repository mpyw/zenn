---
title: "MySQL/Postgres におけるトランザクション分離レベルと発生するアノマリーを整理する"
emoji: "🧱"
type: "tech"
topics: ["database", "transaction", "isolation", "postgresql", "mysql"]
published: true
---

# 読者対象

- ANSI 定義の古典的なトランザクション分離レベルとアノマリーは概ね理解している
- MySQL/Postgres では理論的な部分がどうなっているのかを知りたい

# 理論面の前提知識

**2022-08-19 追記:**<br>社内勉強会向けのスライドを作成しました。先にスライドを見てから，引用文献およびこの記事を読むと理解が深まると思います。

https://speakerdeck.com/mpyw/postgres-niokerutoranzakusiyonfen-li-reberu

----

まず ANSI 定義の古典的な定義を聞いたことが無い方は，以下のリンクを参照されたい。 ANSI 定義に対応する解説はこれらのサイト以外にもたくさんあるため，自分にとって読みやすいと感じる情報をあたってほしい。（既に熟知されている方は十分）

https://itmanabi.com/transaction-isolation/

https://itsakura.com/sql-isolation

https://qiita.com/song_ss/items/38e514b05e9dabae3bdb

https://qiita.com/hatsu/items/4e699ad50651a6a30407

次点で読んでいただきたいのが， [@kumagi](https://zenn.dev/kumagi) さんの以下の記事。古典的には 4 つの分離レベルと 3 つのアノマリーだけで説明されていたものの，不十分であることが学術的に指摘され，解像度を上げようとする流れが後になって起こってきた。

https://qiita.com/kumagi/items/1dc1a91ec007365ac694

https://qiita.com/kumagi/items/5ef5e404546736ebac49

なお上の記事中で言及されている 2-Phase Locking (2PL) に関する説明としては，本人の解説記事および英語版の Wikipedia が分かりやすい。

https://qiita.com/kumagi/items/d3c671ddd1aa5648dd91

https://en.wikipedia.org/wiki/Two-phase_locking

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

ANSI 定義より後に新しく登場したものは太字で表現する。ANSI 定義では読み取りの不整合にしか着目されていなかったが，新しい定義では更新競合と直列化異常にも関心が広げられている。更に定義上，実際に発生はしないものの，書き込み不整合を Dirty Write として据えることになった。

| グルーピング  | 現象                                                        |
|:--------|:----------------------------------------------------------|
| 書き込み不整合 | **Dirty Write**                                           |
| 読み取り不整合 | Dirty Read<br>Fuzzy Read<br>Phantom Read<br>**Read Skew** |
| 更新競合    | **Cursor Lost Update**<br>**Lost Update**                 |
| 直列化異常   | **Write Skew**<br>**Observe Skew**                        |

以下では，それぞれのアノマリーがどういった現象を指しているのかを説明する。

| 現象                                      | 意味                                                                                                                                                    |
|:----------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------|
| Dirty Write                             | 他のトランザクションでコミットされていない変更を上書きしてしまう                                                                                                                      |
| Dirty Read                              | 他のトランザクションでコミットされていない変更を参照してしまう                                                                                                                       |
| Fuzzy Read<br>Phantom Read<br>Read Skew | 他のトランザクションでコミットされた変更を参照してしまう                                                                                                                          |
| Cursor Lost Update                      | 他のトランザクションで **Locking Read によって読み取られているにも関わらず**，<br>コミットされた変更を上書きしてしまう                                                                                |
| Lost Update                             | 他のトランザクションでコミットされた変更を上書きしてしまう                                                                                                                         |
| Write Skew                              | トランザクション $T_1$ が読み取った $x$ を使って $y$ の変更するとき，<br>トランザクション $T_2$ が読み取った $y$ の値を使って $x$ を変更してしまい，<br>**すれ違いざまに相手の変更前の値に依存した更新を行ってしまう**                    |
| Observe Skew                            | 2 つだけであれば直列化可能であったはずのトランザクション $T_1$ $T_2$ の<br>**途中状態**を $T_3$ が観測することによって **循環参照** が発生し，観測者を含めた<br>**$T_1$ $T_2$ $T_3$ の並び順を矛盾なく確定させることができなくなってしまう** |

- MySQL/Postgres とも， **MVCC (Multi Version Concurrency Control)** が採用されているがゆえに Fuzzy Read と Phantom Read はセットで考えることができ，新規・更新・削除の区別を意識する機会は少ない。ここでは簡単のため 1 つにまとめる。
- 2 値の読み取り整合性違反となる Read Skew については，1 つの値を複数回読み取る際の不整合である Fuzzy Read と現象の内容が似ている。これも簡単のために 1 つにまとめる。
- Observe Skew については Read Only Anomaly, Read Only Skew, Batch Processing などさまざまな呼称が存在するが，どれも意味が伝わりにくいので，この記事で Observe Skew という名称を新しく提案する。（論文の中で提案されているものではない）

#### MVCC

発行する SQL 文の種類によって，スナップショットと現在のデータ本体 (Current) のどちらを参照するかが異なっている。

| 文法                               | アクション           | 参照先          | ロック       |
|:---------------------------------|:----------------|:-------------|:----------|
| `SELECT`                         | Consistent Read | **Snapshot** | -         |
| `SELECT ... FOR SHARE`           | Locking Read    | Current      | Shared    |
| `SELECT ... FOR UPDATE`          | Locking Read    | Current      | Exclusive |
| `INSERT`<br>`UPDATE`<br>`DELETE` | Write           | Current      | Exclusive |

- **一貫性読み取り (Consistent Read)** では，最新のデータ本体に依存しないある時点での **スナップショット** をロックせずに取得する。
- **ロック読み取り (Locking Read)** では，最新のデータ本体をロックして取得する。

## MySQL

#### Anomaly

| 現象＼分離レベル                                | READ<br>UNCOMMITTED | READ<br>COMMITTED | **REPEATABLE READ**<br>**[Default]**  | SERIALIZABLE |
|:----------------------------------------|:-------------------:|:-----------------:|:-------------------------------------:|:------------:|
| Dirty Write                             |          ✅          |         ✅         |                   ✅                   |      ✅       |
| Dirty Read                              |          ❌          |         ✅         |                   ✅                   |      ✅       |
| Fuzzy Read<br>Phantom Read<br>Read Skew |          ❌          |         ❌         | 🔺<br>**Broken on**<br>**Mixed Read** |      ✅       |
| Cursor Lost Update                      |          ✅          |         ✅         |                   ✅                   |      ✅       |
| Lost Update                             |          ❌          |         ❌         |                   ❌                   |      ✅       |
| Write Skew                              |          ❌          |         ❌         |                   ❌                   |      ✅       |
| Observe Skew                            |          ❌          |         ❌         |                   ❌                   |      ✅       |

ここでの Mixed Read とは，  Consistent Read と Locking Read/Write の混在を指す。

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

- ロック処理は全て **悲観的制御** である。
- 分離レベルによらず，トランザクションは常に **後勝ち** で処理される。悲観ロックにより競合したトランザクションは，先行トランザクションがコミットするまで待機させれ，それに続いて後続トランザクションが上書きするように処理を続行する。

そのほか，トランザクション分離レベルごとに以下のような特徴がある。

- `READ COMMITTED` 以下では，
  - 毎回の操作で最新状態に基づいたスナップショットを取得する。
- `REPEATABLE READ` 以上では，
  - レコードロックでは担保できない現在レコードが存在しない範囲のロックに関しては， **ギャップロック** で解決している。
  - **`SELECT` 実行で取得したスナップショットバージョン** を，自身が変更しない限りはトランザクションの終了まで保持し続ける。同じクエリに対しては同じ結果が保証される。
  - もし **`BEGIN` の代わりに `START TRANSACTION WITH CONSISTENT SNAPSHOT` を使用すると，トランザクション開始と同時にスナップショットを取得する。** 但し，スナップショット取得後の動作に関しては通常の `REPEATABLE READ` との差異は無い。即ち， 論文で提唱されている SNAPSHOT ISOLATION 分離レベルを満たしているわけではなく，この点は後述されるように Postgres の `REPEATABLE READ` とは明確に異なる。
- `SERIALIZABLE` では， 
  - **Consistent Read はすべて Locking Read に変換される** ため，実質的に存在しない。
  - Lost Update, Write Skew, Observe Skew という直列化異常を全て防ぐことができる。逆に言えば，これらを防ぎたければ Locking Read または `SERIALIZABLE` 分離レベルを使用しなければならない。

:::message alert
##### 分離レベルのダウングレードに注意！
**`REPEATABLE READ` 以上でも，Locking Read/Write では `READ COMMITTED` でロックを取る相当の動作になってしまう仕様となっている。**
- Consistent Read だけをしている限りでは， MVCC によりスナップショットバージョンが固定されているので， 3 種の読み取り不整合はいずれも発生しない。
- Locking Read/Write だけをしている限りでは，  **レコードロック** が Fuzzy Read と Read Skew を防ぎ， **ギャップロック** が Phantom Read を防ぐため， 3 種の読み取り不整合および Lost Update はいずれも発生しない。
- **Consistent Read と Locking Read/Write の間に整合性はないため， これらの併用で不整合が全て起こる可能性がある。** Consistent Read では現れなかった変更が， Locking Read/Write で現れてしまう場合がある。これは複数回実行した場合も該当する。例えば，以下のような状況があり得る。

```sql
BEGIN;
SELECT COUNT(*) FROM example; -- 100
SELECT COUNT(*) FROM example FOR UPDATE; -- 101
SELECT COUNT(*) FROM example; -- 100
SELECT COUNT(*) FROM example FOR UPDATE; -- 101
```
:::

:::message
##### ギャップロックの弊害
`REPEATABLE READ` 以上では，ギャップロックがあることによってデッドロックが発生する機会が増加する弊害がある。 **レコードロックの空振りがギャップロックを引き起こすこともある。** またギャップロックの実装上の都合で存在する **ネクストキーロック** は，直感的なロック範囲よりも広い範囲をロックしてしまうことがある。

- [ネクストキーロックとは | ソフトウェア雑記](https://softwarenote.info/p1067/)
:::

https://dev.mysql.com/doc/refman/8.0/ja/innodb-transaction-isolation-levels.html

#### 総評

MySQL は MVCC を採用しつつも，基本的な戦略を **「悲観的制御」** **「後勝ちトランザクション」** に寄せているシンプルな RDBMS だと言える。 Locking Read で分離レベルをダウングレードさせるなど，思い切った設計にしている面はあるが，それらを受け入れた上で気をつけて使いたい。 `SERIALIZABLE` は悲観ロック処理が重すぎてほぼ実用性なし。

## Postgres

#### Anomaly

|                現象＼分離レベル                 | **READ COMMITTED**<br>**[Default]** |                  REPEATABLE READ                  |                   SERIALIZABLE                    |
|:---------------------------------------:|:-----------------------------------:|:-------------------------------------------------:|:-------------------------------------------------:|
|               Dirty Write               |                  ✅                  |                         ✅                         |                         ✅                         |
|               Dirty Read                |                  ✅                  |                         ✅                         |                         ✅                         |
| Fuzzy Read<br>Phantom Read<br>Read Skew |                  ❌                  |                         ✅                         |                         ✅                         |
|           Cursor Lost Update            |                  ❌                  |                         ✅                         |                         ✅                         | 
|               Lost Update               |                  ❌                  | ✅<br>**Concurrent Update**<br>**Error Detection** | ✅<br>**Concurrent Update**<br>**Error Detection** |
|               Write Skew                |                  ❌                  |                         ❌                         | ✅<br>**R/W Dependencies**<br>**Error Detection**  |
|              Observe Skew               |                  ❌                  |                         ❌                         | ✅<br>**R/W Dependencies**<br>**Error Detection**  |

#### Locking

| アクション＼分離レベル        | **READ COMMITTED**<br>**[Default]** |    REPEATABLE READ    |              SERIALIZABLE               |
|:-------------------|:-----------------------------------:|:---------------------:|:---------------------------------------:|
| Consistent Read    |                  -                  |           -           |             **SIRead ロック**              |
| Locking Read/Write |               レコードロック               | レコードロック<br>**更新競合検査** | レコードロック<br>**更新競合検査**<br>**SIRead ロック** |

:::message
##### SIRead ロックとは？
- **SIRead ロック (Snapshot Isolation Read ロック)** とは，直列化異常検知のための **楽観ロック** を指す。 `SELECT` したデータを SIRead ロックと称して検知のためにマーキングし，後から参照関係をトラバーサルできるようにしておく。
- SIRead ロックを取り入れた Snapshot Isolation (SI) は，  **SSI (Serializable Snapshot Isolation)** と呼ばれる。
:::

:::message
厳密には「更新競合検査」はロック処理に分類すべきではないが，ここでは SIRead ロックとまとめて，楽観ロックに準じた動きをさせるものとして扱う。
:::

#### Errors

| 原因     | メッセージ例                                                                                                                                                                                                             |
|:-------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| デッドロック | ERROR:  deadlock detected<br>DETAIL:<br>Process 37542 waits for ShareLock on transaction 2995;<br>blocked by process 37541.<br>Process 37541 waits for ShareLock on transaction 2996;<br>blocked by process 37542. |
| 更新競合   | ERROR:  could not serialize access due to concurrent update                                                                                                                                                        |
| 直列化異常  | ERROR:<br>could not serialize access due to read/write dependencies among transactions<br>DETAIL:<br>Reason code: Canceled on identification as a pivot, during commit attempt.                                    |

#### 解説

Postgres は以下のような特徴を持つ。

- 悲観的制御はレコードロックのみに抑え，それ以外は軽量な楽観的制御で実現するために **更新競合検査** と **SIRead ロック** を導入している。 
- 悲観的制御で競合したトランザクションは，**後勝ち** で処理される。先行トランザクションがコミットするまで待機させれ，それに続いて後続トランザクションが処理を続行する。
- 楽観的制御で競合したトランザクションは，**先勝ち** で処理される。先行トランザクションが競合するデータをコミットしている場合，後続トランザクションは中止される。

楽観的制御については，表面的な動きだけに着目するのであれば， **「トランザクション開始時から全テーブル・全レコードに対して楽観ロックによるバージョンコントロールを開始する」** と考えても差し支えない。そのほか，トランザクション分離レベルごとに以下のような特徴がある。

- `READ COMMITTED` 以下では，
  - 毎回の操作で最新状態に基づいたスナップショットを取得する。
- `REPEATABLE READ` 以上では，
  - **トランザクション開始と同時に** スナップショットを取得し，自身が変更しない限りはトランザクションの終了まで保持し続ける。
  - レコードロックでは担保できない現在レコードが存在しない範囲のロックに関しては， **更新競合検査** で解決している。更新しようとした内容が更新競合検査に引っかかった場合，トランザクションを更新競合エラーとして中止する。
  - これら 2 つの特性があることにより， Postgres の `REPEATABLE READ` は論文で提唱されている SNAPSHOT ISOLATION 分離レベルも満たしていると言える。
- `SERIALIZABLE` では，
  - レコードロックと更新競合検査でも検知できない直列化異常に関しては， **SIRead ロック** で解決している。更新しようとした内容が SIRead ロックに引っかかった場合，トランザクションを直列化異常エラーとして中止する。
  - Lost Update, Write Skew, Observe Skew という直列化異常を全て防ぐことができる。逆に言えば，これらを防ぎたければ Locking Read または `SERIALIZABLE` 分離レベルを使用しなければならない。

:::message
##### 更新競合検査および SIRead ロックの弊害
- `REPEATABLE READ` 以上では，更新競合検査によってエラーが発生する機会が増えるので，キャッチした上でリトライするなど適切にエラーハンドリングする必要がある。
- 特に `SERIALIZABLE` では SIRead ロック固有のチューニング知識も必要とされ，何も考えずに利用していきなり性能を発揮できるわけではない。詳しくは以下の公式マニュアルの下部を参照。
:::

https://www.postgresql.jp/docs/12/transaction-iso.html

#### 総評

Postgres は `READ COMMITTED` までは MySQL に似た動きをする一方， `REPEATABLE READ` 以上では悲観的制御に加えて **楽観的制御** の仕組みも RDBMS 側に取り入れている。 `SERIALIZABLE` も実用的に使用できる。但しチューニングのために結局気をつけることが増える上に，高頻度な更新処理には適さないため，活躍できるシーンは限られると考えられる。

# どのトランザクション分離レベルを選択すればよいか？<br>ロック戦略はどうすればよいか？

以下は個人的な意見。

## 共通

- 基本的には **`READ COMMITTED`** をベースにした上で **Locking Read** を活用せよ。通常は **`NO WAIT`** オプションを付与してブロッキングを回避し，即時失敗させるとよい。
- Locking Read は空振りするとレコードロックが取得できないので，その懸念がある場合は **アドバイザリーロック** の併用を検討するとよい。

https://zenn.dev/mpyw/articles/rdb-advisory-locks

## MySQL

- デフォルトの `REPEATABLE READ` から `READ COMMITTED` に変更して使用せよ。
- もし `REPEATABLE READ` のまま使用する場合，
  - **ギャップロック** に注意せよ。 **レコードロックの空振り** で意図せず発生させてしまわないように注意。
  - **Consistent Read と Locking Read/Write の混在** に注意せよ。一貫性が欲しい部分でそれらを併用してはならない。
- `SERIALIZABLE` は並列実行性が著しく落ちるため，実用性は薄い。
- `READ UNCOMMITTED` の出番は基本的には無い。

:::message
##### バイナリログ形式に由来する分離レベルの制約
- オンプレミスの環境では，レプリケーションを高速化するためにバイナリログフォーマットを，デフォルトの物理レプリケーション方式の `ROW` から論理レプリケーション方式の `STATEMENT` に切り替えて使用されている場合がある。 **`STATEMENT` である場合は，トランザクション分離レベルを `REPEATABLE READ` 以上にしなければならない制約がある。**
- 独自フォーマットで物理レプリケーションを行っている AWS Aurora 等の場合は `READ COMMITTED` でもレプリケーションが高速に実現されているため，レプリケーション高速化目的でわざわざ `REPEATABLE READ` を選択する必要はない。
:::

## Postgres

- デフォルトの `READ COMMITTED` のまま使用せよ。
- 競合頻度が低い場合， `REPEATABLE READ` 以上で観測される **楽観的制御** の振る舞いに身を任せたほうがパフォーマンスが出る場合があると考えられる。トレードオフとして，リトライもしくはリトライを促すための処理をアプリケーション側で書く必要となる。
  - 業務システムの管理画面等では多少活躍の機会はあるか？と思われたが， `NO WAIT` を付ければそれだけでロック読み取りの欠点であるブロッキングは排除できるので，あまり明確な優位性は無いと感じられる。
  - 特に `SERIALIZABLE` についてはチューニングの知識も求められるため，インフラも含めるとかえって学習コストが上がってしまう懸念がある。無理をして使うよりは，素直に `READ COMMITTED` で Locking Read を用いるほうが汎用性は高い。

:::message
##### 更新競合検査のユースケース
唯一 `REPEATABLE READ` の更新競合検査が明確に適していると言えるのが， **失敗する可能性が高い複数のトランザクションが並行処理され， 1 人だけが勝ち残って更新をコミットすればよい** とき。 

```yaml
# Locking Read で読み込んだあと， 抽選で勝ち残った場合は結果をコミット
T1: LockingRead---Rollback
T2:                       LockingRead---Commit
T3:                                           LockingRead---Rollback
```

```yaml
# Consistent Read で読み込んだあと， 抽選で勝ち残った場合は結果をコミット
T1: ConsistentRead---Rollback
T2: ConsistentRead---Commit
T3: ConsistentRead---Error
```

ブロッキング（`NO WAIT` の場合もリトライ）が発生しない点で明らかに後者に分がある。ちょうどアプリケーションでも楽観ロックを実装したくなる場面だが，この仕組みに乗っかっておけば自前で実装せずに済む点が大きなアドバンテージとなるだろう。
:::

# コラム

:::message
##### 「ネクストキーロック」という言葉の定義

MySQL でファントムリードを防ぐために実装されている仕組みがギャップロックであり，その実装都合で存在するものがネクストキーロックであるとした。ところが，ネクストキーロックという言葉が何を指しているのか，文脈によって意味が異なる場合がある。

B-Tree インデックスが張られているテーブルに関して 10, 20, 30 というレコードが存在し，レコード間のギャップが `-----` の部分に存在していると仮定する。

```yaml
[10]-----[20]-----[30]
```

これらのレコードに対して，以下のクエリを発行する。

```sql
SELECT * FROM example WHERE id BETWEEN 11 AND 15 FOR UPDATE
```

ファントムリードを防ぐために必要となるのは，本来はクエリで選択された 11〜15 の範囲のロックのみで済むはずだが，実際には 11〜20 の範囲がロックされてしまう。このうち，どの部分をネクストキーロックと呼ぶべきだろうか？以下の表をもとに考察を行う。


|     | ロック範囲              | 説明                              |
|:---:|:-------------------|:--------------------------------|
|  X  | 11, 12, 13, 14, 15 | ファントムリードを防ぐために必須となるギャップロック      |
|  Y  | 16, 17, 18, 19     | InnoDB の B-Tree 実装上必要になるギャップロック |
|  Z  | 20                 | InnoDB の B-Tree 実装上必要になるレコードロック |

直感的には，「ネクストキーのロック」つまりは**隣のキーのロック**であると解釈し， Z のレコードロックだけをネクストキーロックと称したくなる。ところが [MySQL 公式マニュアル](https://dev.mysql.com/doc/refman/5.6/ja/innodb-record-level-locks.html) の中では

> ネクストキーロックは、インデックスレコードロックと、そのインデックスレコードの前のギャップに対するギャップロックとを組み合わせたものです。

と明確に記されており，**ギャップロックも含めた** X+Y+Z 全体を **総称としてネクストキーロックと呼んでいる**。これに関しては，以下の資料に詳説がある。

- [MySQLのロックについて - SH2の日記](https://sh2.hatenablog.jp/entry/20140914)

> ひとつ先のインデックスレコードまでロックするのもネクストキーロックと呼ぶし、レコードロックとその直前のギャップロックの組み合わせもネクストキーロックと書いてるし、議論するときにはどちらの意味で使ってるのか文脈読み取れる社会性が必要そうです

> 検索条件に合致したレコードに対してではなく、走査したレコードに対してロックを取得するというところが特徴です。 InnoDB はあくまで MySQL のストレージエンジンであり、検索時にすべての検索条件を把握できているわけではないことが要因の一つだと考えられます。

要点を筆者なりに整理すると，以下のようになる。

- ファントムリードを防ぐために求められるのはギャップロックのみである，という認識は依然として正しい。
- InnoDB は B-Tree インデックス上で走査したレコードに対してロックを取ることで設計が単純化されている。そして範囲検索時に B-Tree インデックス上に閾値と完全一致する値が存在しないときは，**「値が最も閾値に近いノード」** の走査で探索が完了されることになる。これが B-Tree の構造上隣に並ぶので，ロックされるときにはネクストキーロックと呼ばれる。つまり，隣のレコードロックはネクストキーロックである，という認識も依然として正しい。
- InnoDB の B-Tree インデックス上で**物理的にネクストキーロックを取得すると**，ネクストキーまでの範囲で一切レコードの挿入・削除ができなくなるので，結果として**論理的にギャップロックが得られる。**

本来ネクストキーロックは単一のレコードロックに過ぎないが， InnoDB の B-Tree インデックスの構造上，ネクストキーロックによってギャップロックが生み出される仕組みになっているため，**このシステムをまとめてネクストキーロックと呼んでしまおう**，というのが解釈の由来であると考えられる。

とはいえ，筆者個人の意見ではあるが，

*「ネクストキーロックのおかげでファントムリードが起こらない」*

は誤解を招きやすい表現ではあるので，論理的な面を語る上ではギャップロックという言葉を使い， B-Tree インデックスの物理的な構造に触れるときのみネクストキーロックという言葉を積極的に使うようにしたいところではある。
:::

# 謝辞

この記事を執筆するにあたり， Twitter で相談に乗っていただいた皆様に感謝いたします。
以下敬称略

- **[@zyake](https://twitter.com/zyake)**
- [@KentarouTakeda](https://twitter.com/KentarouTakeda)
- [@soudai1025](https://twitter.com/soudai1025)
- [@sji_ch](https://twitter.com/sji_ch)
- [@hmatsu47](https://twitter.com/hmatsu47)
- [@utsushiiro](https://twitter.com/utsushiiro)
- [@neko_han25](https://twitter.com/neko_han25)
