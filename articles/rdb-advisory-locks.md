---
title: "排他制御のためだけに Redis 渋々使ってませんか？データベース単独でアドバイザリーロックできるよ！"
emoji: "🔒"
type: "tech"
topics: ["postgresql", "postgres", "mysql", "database", "redis"]
published: true
---

# 背景

以前， Qiita で以下の記事を投稿した。今回の議題に直接的な関係はないが，関連している部分があるため引用する。

https://qiita.com/mpyw/items/14925c499b689a0cbc59

> MySQL/Postgres とも，
>
> - MVCC アーキテクチャの恩恵で， `SELECT` と `UPDATE` は基本的には競合しない。
> - **単一レコードのシンプルな `UPDATE` でも排他ロックされ**，排他ロック中のレコードへの `UPDATE` での変更操作は **トランザクション分離レベルによらず** ブロックされる。**`UPDATE` 文に含まれる `WHERE` 句での検索もブロックされ**，これはブロックされない `SELECT` による検索とは別扱いになる。
> - **但し `UPDATE` 文の `WHERE` 句上で，更新対象をサブクエリの `SELECT` から自己参照している場合は例外。トランザクション分離レベルを `REPEATABLE READ` 以上にして，競合エラーからの復帰処理を書かなければならない。**
>
> Postgres に関しては，
>
> - `REPEATABLE READ` 以上では， MySQL よりも積極的・予防的に競合エラーを起こすようになっている。上記のように `WHERE` 句に含まれるサブクエリの `SELECT` から自己参照が発生しない場合， **`READ COMMITTED`** にしておくのが最適解。

両データベースとも，書き込み処理競合時， `REPEATABLE READ` ではデッドロックを含むエラーが発生する前提の設計になっており，リトライすることが求められる。一方でエラーを絶対に回避したい場合は，

https://zenn.dev/shuntagami/articles/ea44a20911b817

> 分離レベルを下げ、ギャップロックを無効化することでデッドロックを回避できたものの、 `SELECT...FOR UPDATE` 句の取得結果が `NULL` であった場合にロックがかけられない（ロックする行がない）

とあるように， 更新時は `READ COMMITTED` でロッキングリードしておくことで対応できるものの， **新規作成時には（先行者のコミット完了まで）ロックする行が存在しない** ことで後続者が素通りしてしまう問題がある。

そこで，新規作成を考慮しなければならない操作対象のリソースの代わりに，存在が保証されている別のリソースをロックするルールにしよう，という戦略を取ることができる。これは **アドバイザリーロック（勧告的ロック）** と呼ばれている。

# アドバイザリーロックの実装手段

引用した記事では， `users` というユーザ情報を格納する汎用的なテーブルをアドバイザリーロックのために使用していた。ところがコメント欄でも指摘があるように，汎用的なテーブルをアドバイザリーロックに流用すると，アプリケーション実装者の

*「とりあえず `users` テーブルをロックしておこう！」*

という愚行により， **「特定ユーザに関連してはいるものの内容的には全く関係のない処理」を無駄に待機させてしまう** ことが起こるかもしれない。そのため，以下のような対応を取らなければならない。

- ロック専用テーブルを予め作っておく
- 新規作成してもアドバイザリーロックできる何らかの手段を使う

また，アドバイザリーロックのライフサイクルとして，以下の 2 つの場合があり得る。

- 単一のセッション・トランザクション内で完結する場合
- 複数のセッション・トランザクション上にまたがる場合

これらに着目しながら，各方式を検討していく。

## データベース組み込みのアドバイザリーロック関数

| 観点              | 要求を満たすか |
|:----------------|:-------:|
| 特定 ID を対象としたロック |    ✅    |
| 耐障害性            |    ✅    |
| クライアントの分散       |    ✅    |
| セッションを超えての継続    |    ❌    |

これが今回最もおすすめしたい方法。

### Postgres の場合

https://www.postgresql.jp/document/13/html/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS

| 関数                              | ロックのスコープ  | 競合時の挙動  |
|:--------------------------------|:---------:|:-------:|
| `pg_advisory_lock`              |   セッション   |   待機    |
| `pg_try_advisory_lock`          |   セッション   |   失敗    |
| `pg_advisory_xact_lock`         | トランザクション  |   待機    |
| **`pg_try_advisory_xact_lock`** | トランザクション  |   失敗    |

**`pg_try_advisory_xact_lock`** は，トランザクション開始直後に使用し，解放はトランザクションの終了に任せるという使い方をする。

```sql
BEGIN;

-- ロックの可否を取得
SELECT pg_try_advisory_xact_lock(...);

-- true で成功したときだけ処理を続行
-- ...
-- ...

COMMIT;
```

以下の理由から，この中では **`pg_try_advisory_xact_lock`** が最も使い勝手がよいと考えられる。

- トランザクションがコミットまたはロールバックされて消滅したとき，自動的にロックが解放されるのはありがたい。
- ロックを待機し続けるよりも，ロックに失敗したときに潔く諦めてエラーを返すほうが負荷がかかりにくい。

:::message alert
PHP のように，データベースコネクションを使い捨てにする動作方式が一般的である言語の場合は全て安全に取り扱えるが，その他の多くの言語は **HTTP リクエストの処理が終わってもコネクションを別ユーザのために再利用する** ことが一般的になっている。そのため，ロックのスコープはトランザクションの範囲にしておいたほうが安全ではあるだろう。
:::

:::message alert
複製された読み取り用レプリカインスタンスで実行してはならないならない。必ず **単一の書き込み用プライマリインスタンス** で実行する。
:::

ところで，関数のシグネチャは以下のようになっている。

```java
pg_try_advisory_xact_lock(key bigint): boolean
pg_try_advisory_xact_lock(key1 integer, key2 integer): boolean
```

*「あれ…文字列渡したいんだけど…？」*

文字列を受け取るようにし， Postgres 側がハッシュ化した値をキーにして管理してくれてもいいような気はするが，最小限と割り切ってこのような実装になっているのだろうか？

https://stackoverflow.com/questions/29353845/how-do-i-use-string-as-a-key-to-postgresql-advisory-lock

こちらの回答に従って， 2 つの使い方を示す。

```sql
SELECT pg_try_advisory_xact_lock(hashtext('任意の文字列'));
SELECT pg_try_advisory_xact_lock('テーブル名'::regclass::integer, 主キーの整数);
```

- **`hashtext()`** で **任意文字列に対応する符号付き 64 ビット整数** が取得できるため，必要な情報を全て文字列内に含ませてからハッシュ化した結果を引数として取ることができる。
  - 汎用性が高いが，衝突が発生する可能性がある。しかし，衝突が発生してもロック取得に失敗する程度であるため，エラー対応処理が書かれていればさほど問題はない。 
- **`::regclass::integer`** で **テーブルに対応する一意な符号なし 32 ビット整数** が取得できるため，これをそのまま第 1 引数に取ることができる。

:::message
32 ビット整数を 2 つ取る方法のほうが **絶対に衝突が発生しない** メリットはある。しかし，主キーが 64 ビット整数や `uuid` である場合， 32 ビット整数の範囲に落とし込まないと使えない。また，KVS のように任意のキーをアプリケーション側で組み立てて取り扱いたいケースのほうが多いと考えられるので，基本的にはハッシュ値の引数を 1 つ取る方に倒しておくほうが良い。
:::

:::message alert
Postgres は，トランザクション実行中に **1 回でもエラーが発生すると，ロールバックされるまで後続の処理がすべてエラーになってしまう**。

- [PostgreSQLのトランザクション制御でさっそくハマった２点 - なからなLife](https://atsuizo.hatenadiary.jp/entry/2019/02/26/142814)

セッションをスコープとしてロックを取得していたとき， **エラー発生時のロックの開放処理をロールバック完了まで待たなければならない** 制約が付いてくるため， Web フレームワークなどでトランザクションが高度に抽象化されている場合に適切に対処するのが少々難しい。以下の Laravel 向けのライブラリでは，この問題に対処している。

- [mpyw/laravel-database-advisory-lock: Advisory Locking Features for Postgres/MySQL on Laravel](https://github.com/mpyw/laravel-database-advisory-lock)
:::

**総評: 唯一，セッション・トランザクション内でしかロックを維持できないという欠点を持つが，その範囲の実装でよければ最も優れた選択肢となる。**

### MySQL の場合

https://dev.mysql.com/doc/refman/8.0/ja/locking-functions.html#function_get-lock

Postgres の特権だと思われたが，なんと MySQL にも存在していた。バージョン 5.7 から使えるようだ。

| 関数                              | ロックのスコープ  |       競合時の挙動        |
|:--------------------------------|:---------:|:-------------------:|
| `GET_LOCK`                      |   セッション   | 指定秒数待機<br>タイムアウトで失敗 |

```java
// 成功時に 1, タイムアウトで 0, エラーで NULL 
GET_LOCK(str text, timeout integer): ?integer
```

最初から文字列を受け取れるぶん Postgres より利便性は高い。しかし， **トランザクション終了時に自動解放する手段が無い** ため， `RELEASE_LOCK()` による明示的なロック開放処理を忘れてはならない。また文字列長が **64 文字** に制限されていることに注意。

:::message alert
バイナリログフォーマットが `STATEMENT` のときにロック処理は安全ではないとマニュアルに記載があるが， MySQL 5.6 → 5.7 のアップグレードでデフォルト値が `ROW` に変更されているため，意図的にこのオプションを変更していない限りは問題ないと考えられる。

- [MySQL :: MySQL 5.6 リファレンスマニュアル :: 5.2.4.2 バイナリログ形式の設定](https://dev.mysql.com/doc/refman/5.6/ja/binary-log-setting.html)
- [MySQL 5.6 | db<>fiddle](https://dbfiddle.uk/?rdbms=mysql_5.6&fiddle=f3c32887ccdf7a17407dc8d709c49707)
- [MySQL 5.7 | db<>fiddle](https://dbfiddle.uk/?rdbms=mysql_5.7&fiddle=f3c32887ccdf7a17407dc8d709c49707)

但しオンプレミスの環境では，パフォーマンスを優先して敢えて `STATEMENT` に切り替えた上で論理レプリケーションを行っている場合がある。独自フォーマットで物理レプリケーションを行っている AWS Aurora 等の場合は問題ないが，一応脳の片隅には置いておきたいところ。

- [MySQLのレプリケーション設定で起きたトラブルの原因とその解決策 - Yahoo! JAPAN Tech Blog](https://techblog.yahoo.co.jp/entry/2020072730014361/)
:::

## Redis

| 観点              |                          要求を満たすか                          |
|:----------------|:---------------------------------------------------------:|
| 特定 ID を対象としたロック |                             ✅                             |
| 耐障害性            | 🔺<br>プロセスが落ちると消える<br>レプリカがあっても伝播にリスクあり<br>分散ロックアルゴリズムが必要 |
| クライアントの分散       |                             ✅                             |
| セッションを超えての継続    |                             ✅                             |

*排他制御といえば Redis！*

…筆者もそう思っていたが，ここにきて認識が揺らいでいる。今まで Redis が落ちない前提で `SET` を使った処理を書いていたが，どうやらそこまで安全とは言えないようだ。

まず最もオーソドックスな，シンプルに `SET` 命令を使う方法を解説する。既に紹介したデータベースのアドバイザリーロック関数を使う方法と異なり， **有効期限を設定してセッションを超えた継続** が可能になっている点がポイント。

https://redis.io/commands/set/

| オプション | 解説                                                                                               |
|:-----:|:-------------------------------------------------------------------------------------------------|
| `NX`  | **存在しない場合のみ作成** を実現できる。返り値は作成した場合は `"OK"`，しなかった場合は `(nil)` となるので，この結果を見ることでロックを取得出来たかどうかの区別ができる。 |
| `EX`  | タイムアウトを設定できる。クライアントが意図せずクラッシュしたときのため，一定時間経過で自動回復出来るようにしておいたほうが無難。                                |

```java
// 新規ロック取得時 
SET キー "所有者" NX EX タイムアウト秒数

// 復帰時 （取得後， 所有者が自分かどうか確認する）
GET キー

// 終了時
DEL キー
```

:::message
`所有者` は，プロセス ID など，リクエストを超えて保持したい処理者を一意に識別できるものを指す。
:::

:::message alert
複製された読み取り用レプリカインスタンスで実行してはならないならない。必ず **単一の書き込み用プライマリインスタンス** で実行する。
:::

Redis において非常によく見られる使い方ではある。しかしこれで万端かと思いきや， **レプリカへの伝播失敗でロック情報が消失するケースがある** という。

https://christina04.hatenablog.com/entry/redis-distributed-locking

対処法として **RedLock** という，複数ノードを用いた分散ロックアルゴリズムが紹介されている。自前で書くのは骨が折れるため，ライブラリを使いたいところ。

:::message
RedLock にも時間経過で意図せずロックが解放される問題点があると記事で言及されているが，ここではガベージコレクションがボトルネックになるような極めて短い時間軸の話をしており，それよりも長い時間軸で余裕を持ってロックを取る場合は大きな問題にはならない。**タイムアウトは十分処理が間に合うように，かつ無駄に長すぎないように，これら 2 点をどちらも気にかけて決定しよう。**
:::

:::message
**2022-07-07 18:45 追記**

- 現在，Redis 社が **[RedisRaft](https://github.com/RedisLabs/redisraft)** という，速さを犠牲にして強整合性を担保する **[Raft Consensus Algorithm](https://raft.github.io/)** を実験的に実装しているそうだ。これが Redis に本採用されれば， RedLock に頼らない選択肢が増えるという。
- AWS の場合， **[MemoryDB](https://aws.amazon.com/jp/memorydb/)** という **[ElastiCache](https://aws.amazon.com/jp/elasticache/)** とは別のフルマネージドな Redis 互換実装が提供されており，こちらは RedisRaft のような強整合性を担保できるようだ。排他制御の用途では間違いなく ElastiCache よりも MemoryDB のほうが適任ではあるので，費用を見積もった上で抵抗がないなら採用を検討してもよいだろう。
:::

**総評: 汎用性が高いが，厳密に安全性を考えると RedLock などのアルゴリズムに基づいた冗長化が必要になり，ややコストが高い。但し，AWS の場合は ElastiCache の代わりに MemoryDB を使えば解決する。また，将来的に RedisRaft が正式導入されればオープンソースな Redis でも解決できるようになる可能性がある。**

## OS のファイルロック

| 観点              |           要求を満たすか            |
|:----------------|:----------------------------:|
| 特定 ID を対象としたロック | 🔺<br>クリーンアップにダウンタイムを設ける必要あり |
| 耐障害性            |      🔺<br>ストレージが 1 箇所       |
| クライアントの分散       |       ❌<br>ストレージが 1 箇所       |
| セッションを超えての継続    |              ✅               |

古くからよく使われていた手法であるが，ファイルを 1 箇所で管理する都合上，クライアントを分散できないため，あまり商業案件での実用性は無い。個人の VPS で分散させずに小規模に運用する場合には依然として使える場合もある。

- [Man page of FLOCK](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/flock.2.html)
- [PHP: flock - Manual](https://www.php.net/manual/ja/function.flock.php)

ファイルのロックに使う `flock` 関数はまさにアドバイザリーロックを提供するものであり，ファイルの編集以外にも流用することができる。更に，ファイルの更新日時やファイルの中身を追加の情報ソースとして利用でき，有効期限等も設定することができるため，自由度は高い部分もある。但し，ダウンタイム無しの **ファイルのクリーンアップ処理が難しい** という大きな欠点を持つ。そのため，クリーンアップ処理が不要な，特定の ID に依存しない排他制御にしか適用が困難である。

:::message alert
ファイルを削除してしまうと inode が変わってしまい， **削除前に `fopen()` したユーザ** と **削除後に `fopen()` したユーザ** が別の inode を参照してしまうため，レースコンディションが発生する。

- [Why flock doesn't clean the lock file? - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/368159/why-flock-doesnt-clean-the-lock-file/368167#368167)
- [FlockLock failed to remove temporary lock file. · Issue #28 · arvenil/mutex](https://github.com/arvenil/mutex/issues/28#issuecomment-186233223)
:::

**総評: 過去の遺物**

## ロック専用テーブル

| 観点              |           要求を満たすか            |
|:----------------|:----------------------------:|
| 特定 ID を対象としたロック | 🔺<br>排他制御したい単位の個数分のインサートが必要 |
| 耐障害性            |              ✅               |
| クライアントの分散       |              ✅               |
| セッションを超えての継続    |              ✅               |

```sql
CREATE TABLE mutex(
    key varchar(64) PRIMARY KEY
);
```

```sql
-- ユーザ 1 人ごとに， 注文操作は重複してできないようにする
INSERT INTO users(id, name) VALUES ('U1', 'Bob'), ('U2', 'Alice');
INSERT INTO mutex(key) VALUES ('user:U1:order'), ('user:U2:order');
```

このように準備しておいた上で，以下のように `SELECT ... FOR UPDATE` を利用する。取得失敗時に即座に終了したい場合は， `NOWAIT` を付けたほうがよい。

```sql
BEGIN;

-- ロックの可否を取得
SELECT * FROM mutex WHERE key = 'user:U1:order' FOR UPDATE NOWAIT;

-- ロックレコードを取得成功したときだけ処理を続行
-- ...
-- ...

COMMIT;
```

:::message alert
複製された読み取り用レプリカインスタンスで実行してはならない。必ず **単一の書き込み用プライマリインスタンス** で実行する。
:::

ロックしたい単位でレコードを事前準備しておく必要性がある点以外は取り立てて問題はない。また応用として，以下のようにセッションを超えた継続を実現することもできる。

```sql
-- Postgres
CREATE TABLE mutex(
    key varchar(64) PRIMARY KEY,
    owner varchar(64) NOT NULL, -- 所有者
    expires_at TIMESTAMPTZ NOT NULL DEFAULT '1970-01-01 00:00:00' -- ロックの有効期限
);

-- MySQL
CREATE TABLE mutex(
    `key` varchar(64) PRIMARY KEY,
    owner varchar(64) NOT NULL, -- 所有者
    expires_at datetime NOT NULL DEFAULT '1970-01-01 00:00:00' -- ロックの有効期限
);
```

```sql
BEGIN;

-- ロックの可否を取得
-- 取得できた場合， expires_at を更新
WITH m AS (
    SELECT key FROM mutex
    WHERE key = 'user:U1:order'
    AND (
        owner = '所有者'       -- 所有者自身による引き継ぎ
        OR expires_at < NOW() -- ロックの有効期限が切れた
    )
    FOR UPDATE NOWAIT
)
UPDATE mutex SET owner = '所有者', expires_at = ...
WHERE key = (SELECT key FROM m);

-- UPDATE が作用したときだけ処理を続行
-- ...
-- ...

COMMIT;
```

**総評: 最も堅実な手段ではあるが，ロック用のレコードを予め据えておくのがとにかく億劫。**

# 応用集

## アドバイザリーロック関数 + テーブルでのロック情報保持

- アドバイザリーロック関数の「セッション越えを出来ない」という弱点
- ロック専用テーブルの「予めレコードを用意しておく必要がある」という弱点

を両方とも克服する手法。 以下に Postgres を想定して記述するが， MySQL でもほぼ同様である。

```sql
BEGIN;

-- 空振りを防ぐためのアドバイザリーロック
-- 成功したときだけ続行
SELECT pg_try_advisory_xact_lock('user:U1:order');

WITH
m_all AS (
    SELECT * FROM mutex
    WHERE key = 'user:U1:order'
    FOR UPDATE NOWAIT -- アドバイザリーロックをしているので必須ではないが，一応付けておく
),
-- 「古いロック」または「引き継ぎ可能なロック」に該当するもの
m_stale AS (
    SELECT * FROM m_all
    WHERE (
        owner = '所有者'       -- 所有者自身による引き継ぎ
        OR expires_at < NOW() -- ロックの有効期限が切れた
    )
)
INSERT mutex(key, owner, expires_at)
SELECT 'user:U1:order', '所有者', ...
WHERE NOT EXISTS(SELECT * FROM m_all) -- ロックが存在せず新規作成される場合
   OR EXISTS(SELECT * FROM m_stale)   -- 古いロックまたは引き継ぎ可能なロックが残っている場合
ON CONFLICT(key) DO UPDATE
SET owner = EXCLUDED.owner, expires_at = EXCLUDED.expires_at 
RETURNING *;

-- INSERT または UPDATE が作用したときだけ処理を続行
-- ...
-- ...

COMMIT;
```

## 空打ちでエラー回避できる `INSERT`  + テーブルでのロック情報保持

```sql
-- Postgres
INSERT INTO mutex(`key`, owner, expires_at)
VALUES ('user:U1:order', '所有者', ...)
ON CONFLICT(key) DO NOTHING;

-- MySQL
INSERT INTO mutex(`key`, owner, expires_at)
VALUES ('user:U1:order', '所有者', ...)
ON DUPLICATE KEY UPDATE `key` = VALUES(`key`);
```

https://www.slideshare.net/ichirin2501/insert-51938787

こちらで紹介されているテクニック。一見 MySQL 専用かと思いきや， Postgres は `INSERT IGNORE` よりもお行儀のいい **`ON CONFLICT(key) DO NOTHING`** を備えているため，バッドノウハウっぽい書き方をしなくても達成できる。

:::message
MySQL が **「`REPEATABLE READ` であっても大丈夫」** というだけで，「`REPEATABLE READ` でなければならない」という訳ではない。
:::

:::message alert
トランザクションが競合した際， MySQL の `REPEATABLE READ` は可能な限り先行者を優先して続行しようとするが， Postgres はどちらも失敗させるという違いがある。そのため， Postgres がこの方法を使う場合は `READ COMMITTED` でなければならない。
:::

大きな欠点として， **`INSERT` でブロッキングが発生する** 点が挙げられる。そのため，後続の `SELECT ... FOR UPDATE` に `NOWAIT` をつけても意味がない。ノンブロッキングに処理できないため，この方法はあまり推奨されない。

以下は MySQL の例

```sql
BEGIN;

-- ON DUPLICATE KEY UPDATE で意味のない更新を書くバッドノウハウ
INSERT INTO mutex(`key`, owner, expires_at)
VALUES ('user:U1:order', '所有者', ...)
ON DUPLICATE KEY UPDATE `key` = VALUES(`key`);

-- ロックの可否を取得
-- 取得できた場合， expires_at を更新
WITH m AS (
    SELECT `key` FROM mutex
    WHERE `key` = 'user:U1:order'
    AND (
        owner = '所有者'       -- 所有者自身による引き継ぎ
        OR expires_at < NOW() -- ロックの有効期限が切れた
    )
    FOR UPDATE
)
UPDATE mutex SET owner = '所有者', expires_at = ...
WHERE `key` = (SELECT `key` FROM m);

-- UPDATE が作用したときだけ処理を続行
-- ...
-- ...

COMMIT;
```

# まとめ

## 基本はアドバイザリーロック関数で済ませるとよい

それぞれの関数には癖があるので，特性を再確認しておく。

```sql
-- Postgres
SELECT pg_try_advisory_xact_lock(hashtext('任意の文字列'));

-- MySQL
SELECT GET_LOCK('任意の文字列', タイムアウト);
```

## セッションを越えてロックを保持したければ，ロック用テーブルを使う

まだレコードが存在していないときの空振り対策として，以下のいずれかを選択する。

- あらかじめレコードを埋めておく
- アドバイザリーロック関数を併用する

:::message alert
Postgres は必ず `READ COMMITTED` でなければならない
:::

但し AWS の場合は， MemoryDB の採用を検討してもよい。

https://aws.amazon.com/jp/memorydb/

従来の Redis では満たせなかった強整合性・耐障害性を担保できているため，予算さえ許せば優秀な選択肢であると考えられる。
