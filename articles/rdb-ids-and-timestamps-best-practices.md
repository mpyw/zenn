---
title: "Postgres と MySQL における id, created_at, updated_at に関するベストプラクティス"
emoji: "🎓"
type: "tech"
topics: ["postgresql", "postgres", "mysql", "database", "rdb"]
published: true
---

# 読者対象

- ある程度データベースに関する知識を持っている，経験年数 1 年以上のバックエンドエンジニア
- 特定のプログラミング言語に依存する部分は含めないため，すべての SQL 使用者を対象とする

また，ゼロからの丁寧な説明というよりは，リファレンス感覚で使える記事という形にまとめる。

# RDBMS の対象バージョン

- PostgreSQL: 9.4 以降
- MySQL: 8.0.28 以降

# `id` （データ型と INSERT 時のデフォルト埋め）

## 導入

一般的に採用されやすいプライマリキー用の値として，以下を考える。

- **連番整数**
  - MySQL では [`AUTO_INCREMENT`](https://dev.mysql.com/doc/refman/8.0/en/example-auto-increment.html)， Postgres では [`SERIAL`](https://www.postgresql.org/docs/current/datatype-numeric.html#DATATYPE-SERIAL) と呼ばれるもの
- **[UUID v1](https://ja.wikipedia.org/wiki/UUID)**: ハードウェアごとにユニークなシーケンシャル値
- **[UUID v4](https://ja.wikipedia.org/wiki/UUID)**: ランダム値
- **[UUID v7](https://asnokaze.hatenablog.com/entry/2021/04/28/030550)**（ドラフト）: タイムスタンプとランダム値の複合

それぞれ，以下のような特徴を持つ。

|          |      連番整数      |                    UUID v1                     |  UUID v4   |                    UUID v7                    |
|:---------|:--------------:|:----------------------------------------------:|:----------:|:---------------------------------------------:|
| スケーラビリティ |      集権的       |                    **分散可能**                    |  **分散可能**  |                   **分散可能**                    |
| 推測可能性    |    容易に推測可能     |                      推測可能                      |   **不可**   |          **ほぼ推測不可**<br>（暗号学的には安全でない）          |
| 時系列ソート   | **アトミックな順序保証** | **フィールドをスワップすれば順序保証可能**<br>（更にハードウェア単位ではアトミック） |     不可     |                **ミリ秒精度で順序保証**                 |
| 衝突可能性    |     **0%**     |          **0%**<br>（MAC アドレスが衝突しない限り）          | **ほぼ 0%**  | **ほぼ 0%**<br>（ミリ秒オーダーの速さで連続発行する場合はやや衝突確率が上がる） |
 | 状態への依存   |     ステートフル     |                     ステートフル                     | **ステートレス** |                  **ステートレス**                   |  

また，（UUID バージョンによらず） UUID 形式の値を取り扱う場合，各 RDBMS によって相性がある。

|                    |                                                      MySQL                                                      |                                              Postgres                                               |
|:-------------------|:---------------------------------------------------------------------------------------------------------------:|:---------------------------------------------------------------------------------------------------:|
| 専用データ型             |                                      なし<br>（`CHAR(36)` か `BINARY(16)` で代替）                                      |                                       **あり**<br>（**`UUID`**）                                        |
| 時系列ソート可能でないキーの性能劣化 |                                                    致命的な問題あり                                                     |                                    極めて大量のデータでは若干の性能低下が起こる可能性がある                                     |
| 値を生成する標準関数         | **[`UUID()`](https://dev.mysql.com/doc/refman/8.0/en/miscellaneous-functions.html#function_uuid)**<br>（UUID v1） | **[`gen_random_uuid()`](https://www.postgresql.org/docs/current/functions-uuid.html)**<br>（UUID v4） |

## 選考基準

まず実装細部には触れずに，選考基準だけを考える。

### Postgres

Postgres は，[ネイティブで UUID 型をサポート](https://www.postgresql.org/docs/current/datatype-uuid.html)している。これは，以下のような特性があることを意味する。

- 内部的には 16 バイトのバイナリ形式で格納されるため，最も空間効率が良い。
- 表層的には `xxxxxxxx-xxxx-xxxx-Nxxx-xxxxxxxxxxxx` の 16 進数表記に見えるため，プログラムから取り扱いやすい。

また， [`gen_random_uuid()`](https://www.postgresql.org/docs/current/functions-uuid.html) という UUID v4 の生成関数も標準で提供されているため，デフォルト値としての自動生成も必要に応じてできる。 Postgres においては，アトミックな順序保証が欲しかったり，インデックスサイズを小さくする必要があったり，MySQL の章にて後述する「クラスターインデックス」を意図的に Postgres で使ったりしない場合は， UUID v4 が最有力な選択肢となるだろう。

基本は `created_at` `updated_at` を併用してソートすべきであるが， `id` 単体での時系列ソートが欲しいときは， UUID v7 や UUID v1 を使うことになる。この場合は， UUID v7 の使用を推奨する。 **UUID v1 は MAC アドレス依存があるが， Docker 環境の中にいる場合は取得できる値が一意であることが保証されにくいためである。**

また後述するように， MySQL ほど致命的な影響が無いものの，数千億・数兆単位のレコードでは，理論上はわずかに影響が出てくる可能性がある。このクラスのデータ量を扱うことが事前に分かっている場合は， UUID v7 がより安全な選択にはなるだろう。

:::message
**選考のポイント**

- 特に拘りが無ければ **UUID v4** を選ぶ。
- 基本は `created_at` `updated_at` を併用してソートすべきであるが， ID でどうしてもソートしたければ **UUID v7** または UUID v1 を選んでもよい。また，数千億・数兆単位のレコードを扱う場合は， UUID v7 のほうが理論上は性能劣化しにくい。
- `SERIAL` は，アトミックな順序保証が欲しいなどの特殊なケースを除いては，それほど積極的に採用する理由はない。
:::

:::message alert
UUID v1 は， Docker 環境の中でクライアントから能動的に発行することを想定する場合は推奨されない。
:::

### MySQL

#### UUID: データ型の検討

MySQL は，ネイティブで UUID 型をサポートしていない。そのため， `CHAR(36)` か `BINARY(16)` を選択することになる。

- `CHAR(36)`
  - `xxxxxxxx-xxxx-xxxx-Nxxx-xxxxxxxxxxxx` の 16 進数表記の文字列としてみた場合， 36 バイト必要になる。表層的には取り扱いやすいが，そのまま `CHAR` として格納すると空間効率が非常に悪い。
- `BINARY(16)`
  - Postgres と同じ効率の格納ができるが， SELECT してきた結果がバイナリデータとなるため，プログラムからそのままでは扱いにくい。加工に一工夫必要となる。

どちらを選択しても，一長一短でそれなりのデメリットが目立つ。但し，生成カラムを活用してバイナリを取り扱いやすくする方法はあるので，それは後述する。

#### UUID: バージョンの検討

MySQL においては， Postgres と異なり，すべてのテーブルにおいてデフォルトで **[クラスターインデックス](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)** というフォーマットが強制される。これが原因となって，**シーケンシャルでない値が連続インサートされたときに大きく性能が劣化することが知られている**。そのため， MySQL で主キーとして UUID を利用する場合，最もよく使われる完全にランダムな UUID v4 を使うことが不利に働く場合がある。

- **[MySQLでプライマリキーをUUIDにする前に知っておいて欲しいこと | Raccoon Tech Blog](https://techblog.raccoon.ne.jp/archives/1627262796.html)**
- **[MySQLとPostgreSQLと主キー - Speaker Deck](https://speakerdeck.com/hmatsu47/mysqltopostgresqltozhu-ki?slide=26)**

これらを踏まえると，最も安牌な選択肢は連番整数となる。次点で UUID v7， UUID v1 となる。 UUID v1 は Postgres の項でも紹介したように， Docker との親和性の悪さの問題があるので注意。

:::message
**選考のポイント**

- 最も簡単な選択肢を採りたい場合， **`AUTO_INCREMENT`** を用いて連番整数を採番する。
- スケーラビリティを鑑みて分散的に ID を発行したい場合， **UUID v7** または UUID v1 を利用する。
  - 取り扱いを優先する場合は `CHAR(36)` 型を利用する。
  - 効率を優先する場合は `BINARY(16)` 型を利用し，取り出してきてから 16 進数表記文字列に変換する。
:::

:::message alert
UUID v1 は， Docker 環境の中でクライアントから能動的に発行することを想定する場合は推奨されない。
:::

## テーブル定義例

次に，実装例としてのテーブル定義を見ていこう。

### Postgres

`SERIAL` を使う場合は省略し， UUID v4 と UUID v7 の例を紹介する。

#### UUID v4

Postgres は基本これ。

```sql
CREATE TABLE users(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid()
);
```

UUID v4 の特徴としてプログラム側で事前に生成しておけるメリットはあるが，省略した場合のデフォルト値が設定されていたほうが使い勝手は良い。書いておくに越したことはない。

#### UUID v7

シーケンシャルであってほしい場合はこちら。 

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE OR REPLACE FUNCTION uuid_generate_v7() RETURNS UUID AS
$$
DECLARE
  ver_rand_var bytea = e'\\000\\000\\000';
  unix_time_ms bytea;
  rand_bytes   bytea;
BEGIN
  unix_time_ms = substring(int8send(floor(extract(epoch from clock_timestamp()) * 1000)::bigint) from 3);
  rand_bytes = gen_random_bytes(3);

  ver_rand_var = set_byte(ver_rand_var, 0, (b'0111'||get_byte(rand_bytes, 0)::bit(4))::bit(8)::int);
  ver_rand_var = set_byte(ver_rand_var, 1, get_byte(rand_bytes, 1));
  ver_rand_var = set_byte(ver_rand_var, 2, (b'10'||get_byte(rand_bytes, 2)::bit(6))::bit(8)::int);

  return substring((unix_time_ms || ver_rand_var || gen_random_bytes(7))::text from 3)::uuid;
END
$$ LANGUAGE plpgsql;
```

```sql
CREATE TABLE users(
    id UUID PRIMARY KEY DEFAULT uuid_generate_v7()
);
```

ユーザ定義関数に関しては， [こちらの Gist](https://gist.github.com/kjmph/5bd772b2c2df145aa645b837da7eca74) の実装を利用したところ，動作することを確認した。コメント欄を見る限り，ベンチマーク的にも大きな問題は無さそうとのこと。こちらに関しては可読性が少し悪いため，常にプログラム側で生成する前提で，デフォルト値は無しにしても良い。

#### UUID v1

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users(
    id UUID PRIMARY KEY DEFAULT uuid_generate_v1()
);
```

上記のように， `uuid-ossp` というモジュールを読み込むことで，ドラフトである v7 を除いて UUID v1, v3, v4, v5 すべてに対応できるようになる。

### MySQL

`AUTO_INCREMENT` を使う場合は省略し， UUID v1 の例を紹介する。 UUID v7 の例を示したかったが， Postgres のような参考実装をまだ見つけられていない。（情報提供求む）

#### UUID v1

[MySQL 8.0.13 からは任意の式がデフォルト値として利用できるようになった](https://stackoverflow.com/a/46134649)ため，当該バージョンである場合はインラインで書くことができる。

:::message alert
MySQL 8.0.12 までは，トリガーを作って対処するしかない。
:::

```sql
-- 16 進数文字列
CREATE TABLE users(
    id CHAR(36) PRIMARY KEY DEFAULT (BIN_TO_UUID(UUID_TO_BIN(UUID(), 1))) CHARACTER SET ascii
);
```

```sql
-- バイナリ
CREATE TABLE users(
    id BINARY(16) PRIMARY KEY DEFAULT (UUID_TO_BIN(UUID(), 1)),
    hex CHAR(36) AS (BIN_TO_UUID(id)) VIRTUAL NOT NULL
);
```

- 共通
  - [`UUID_TO_BIN()`](https://dev.mysql.com/doc/refman/8.0/en/miscellaneous-functions.html#function_uuid-to-bin) の *swap_flag* を有効にすることで，タイムスタンプ部の `LOW-MID-HIGH` を `HIGH-MID-LOW` に置き換えている（「秒・分・時」のようなフォーマットを「時・分・秒」に置き換えているイメージに近い）。こうすることによって，時系列ソートできるようにデータが加工されている。
  :::message alert
  変換操作により，本物の UUID としてのフォーマットは崩されてしまうことは避けられない。UUID っぽい見た目をした UUID ではない何か，ということになる。
  :::
- `CHAR(36)`
  - 文字セットを局所的に `ascii` にすることにより，デフォルトに設定されているであろう `utf8mb4` よりも消費バイト数を小さくしている。
- `BINARY(16)`
  - **計算結果が保存されない（`STORED` ではなく `VIRTUAL` である）生成カラム** を定義しておくと， SELECT でデータ取得したときのみ計算コストが発生するため，今回の用途に適している。
  :::message alert
  `VIRTUAL` である生成カラムは，結果の取得のみに用い，絞り込み条件の対象としてはならない。必ず，入力値をバイナリ変換して使うようにする。
  ```sql
  SELECT * FROM users WHERE id = UUID_TO_BIN('xxxxxxxx-xxxx-xxxx-Nxxx-xxxxxxxxxxxx')
  ```
  :::

:::details コラム: ULID と UUID v7

UUID v7 が提唱されるよりも前に，時系列ソート可能なランダム値の実装がいくつか考えられてきたが，その 1 つに **[ULID](https://github.com/ulid/spec)** がある。エンコードされている状態の見た目は大きく UUID と異なるが，バイナリフォーマットを覗いてみるとその実態は UUID v7 に極めて近い。

```
UUID v7

0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           unix_ts_ms                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          unix_ts_ms           |  ver  |       rand_a          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|var|                        rand_b                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            rand_b                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

- ミリ秒精度タイムスタンプ 48 ビット
- 乱数 74 ビット
- バージョン 4 ビット
- バリアント 2 ビット
```

```
ULID

0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      32_bit_uint_time_high                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     16_bit_uint_time_low      |       16_bit_uint_random      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       32_bit_uint_random                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       32_bit_uint_random                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

- ミリ秒精度タイムスタンプ 48 ビット
- 乱数 80 ビット
```

この特徴のため， ULID は UUID フォーマットに変換し， UUID v7 っぽいものとして使用することができる。[mpyw/uuid-ulid-converter](https://github.com/mpyw/uuid-ulid-converter) のように，フォーマットを相互変換するライブラリを作ることも可能である。

:::message alert
変換操作では，本物の UUID としてのフォーマットに従ったものは作れない。UUID っぽい見た目をした UUID ではない何か，ということになる。
:::

# `created_at` `updated_at` （データ型と INSERT 時のデフォルト埋め）

## 導入

作成日時を記録する，日付時刻系のデータ型として一般的なものを考える。

|                                          | `DATETIME`<br>(MySQL)              | `TIMESTAMP`<br>(MySQL)             | `TIMESTAMP`<br>(Postgres)    | `TIMESTAMPTZ`<br>(Postgres)  |
|:-----------------------------------------|------------------------------------|:-----------------------------------|:-----------------------------|:-----------------------------|
| 日付時刻文字列 INSERT での<br>タイムゾーン表現 `+0900` など | 無視                                 | **考慮**                             | 無視                           | **考慮**                       |
| SELECT でクライアント時刻を考慮して<br>日付時刻文字列表示       | 無視                                 | **考慮**                             | 無視                           | **考慮**                       |
| 表現可能な区間                                  | **西暦 1000 年 〜 西暦 9999 年**          | 西暦 1970 年 〜 西暦 2038 年              | **紀元前 4713 年 〜 西暦 294276 年** | **紀元前 4713 年 〜 西暦 294276 年** |
| 精度                                       | デフォルトでは秒<br>（**オプション指定でマイクロ秒まで可**） | デフォルトでは秒<br>（**オプション指定でマイクロ秒まで可**） | **マイクロ秒**                    | **マイクロ秒**                    |

:::message alert
MySQL の `TIMESTAMP` には「2038 年問題」と呼ばれる有名な問題がある。
:::

## 選考基準とテーブル定義例

### Postgres

基本的に， タイムゾーン付きのタイムスタンプである **`TIMESTAMPTZ`** 1択と考えて良い。

```sql
CREATE TABLE users(
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP -- トランザクション中は開始時刻に固定される
);
CREATE TABLE users(
    created_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp() -- 常に現在時刻を取る
);

-- TIMESTAMPTZ を省略せずに丁寧に書く場合
CREATE TABLE users(
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

:::message alert
Postgres においては， `CURRENT_TIMESTAMP` `current_timestamp()` `now()` などは全て**トランザクション開始時刻**を返すようになっている。トランザクションに左右されない MySQL のような仕様にしたい場合は，代わりに [`clock_timestamp()`](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-CURRENT) という関数を使う必要がある。必要に応じて使い分けたい。
:::

### MySQL

Postgres と同様に，タイムゾーンを考慮してくれる `TIMESTAMP` を使いたいが， 2038 年で頭打ちになるデータ型を 2022 年の今使うのはかなり悩むところ。よって，

- `DATETIME` または `DATETIME(3)`（ミリ秒精度） または `DATETIME(6)`(マイクロ秒精度）

としてタイムゾーンは捨てた上で日付で格納するのが苦肉の策。または， [MySQL 8.0.28 からはタイムスタンプと日付の変換関数が西暦 3000 年までのデータを取り扱えるようになった](https://dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html#function_unix-timestamp)ため，

- `BIGINT UNSIGNED` または `DECIMAL(65, 3)`（ミリ秒精度） または `DECIMAL(65, 6)`（マイクロ秒精度）

としてタイムスタンプを数値で格納するのも 1 つの戦略である（後者は UTC 基準であることを明確にしたい狙いが大きい）。後者の場合，先に UUID バイナリに関して説明した通り， `VIRTUAL` な生成カラムを組み合わせてもよい。

:::message alert
MySQL 8.0.27 までは，西暦 2038 年以降のデータの計算結果は 0 になってしまう。
:::

:::message alert
MySQL 8.0.28 以降も， `TIMESTAMP` 型のフィールドの範囲は西暦 2038 年から拡充されていない。
:::

```sql
-- DATETIME（マイクロ秒精度）
CREATE TABLE users(
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
);

-- タイムスタンプ変換した DECIMAL（マイクロ秒精度）
CREATE TABLE users(
    created_at DECIMAL(65, 6) NOT NULL DEFAULT (UNIX_TIMESTAMP(CURRENT_TIMESTAMP(6))),
    tz_created_at DATETIME(6) AS (FROM_UNIXTIME(created_at)) VIRTUAL NOT NULL
);
```

# `updated_at` （UPDATE 時のデフォルト埋め）

## 導入

データ型としては `created_at` と全く同じだが，「行が更新されたときに埋める」を実現するための方法が， MySQL と Postgres で大きく異なる。

## 選考基準とテーブル定義例

### MySQL

MySQL は， `ON UPDATE CURRENT_TIMESTAMP` という便利なシンタックスを備えており，これにより更新された場合のみ自動で埋めることが簡単にできる。

```sql
CREATE TABLE users(
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6)
);
```

:::message alert
`updated_at` の値を明示的に更新するクエリが流れてきたときは， `ON UPDATE CURRENT_TIMESTAMP` の内容は適用されない。
:::

但し， `DEFAULT` と違って任意の式を記述することはできないので， `CURRENT_TIMESTAMP` を代入可能な `TIMESTAMP` `DATETIME` 以外の場合はトリガーで対処するしかない。 MySQL は Postgres と異なり，トリガーの再利用が不可能なので，極めて冗長な記述が必要になってしまう。 `BIGINT` や `DECIMAL` を採用した場合は，自動更新は諦めたほうが賢明だろう。

### Postgres

実は今回の記事でメイントピックとしたかった内容がこれ。 Postgres に `ON UPDATE CURRENT_TIMESTAMP` という機能は存在しないので，どんな場合でもトリガーを書かなければならない。トリガーの再利用ができる点はマシだと思いたいところ。

> `updated_at` の値を明示的に更新するクエリが流れてきたときは， `ON UPDATE CURRENT_TIMESTAMP` の内容は適用されない。

と MySQL の項で述べたが，これを再現するためには[トリガーが 3 つも必要になってしまう](https://stackoverflow.com/a/8762116)。

```sql
CREATE FUNCTION refresh_updated_at_step1() RETURNS trigger AS
$$
BEGIN
  IF NEW.updated_at = OLD.updated_at THEN
    NEW.updated_at := NULL;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
    
CREATE FUNCTION refresh_updated_at_step2() RETURNS trigger AS
$$
BEGIN
  IF NEW.updated_at IS NULL THEN
    NEW.updated_at := OLD.updated_at;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE FUNCTION refresh_updated_at_step3() RETURNS trigger AS
$$
BEGIN
  IF NEW.updated_at IS NULL THEN
    NEW.updated_at := CURRENT_TIMESTAMP;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

```sql
CREATE TABLE users(
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TRIGGER refresh_users_updated_at_step1
  BEFORE UPDATE ON users FOR EACH ROW
  EXECUTE PROCEDURE refresh_updated_at_step1();
CREATE TRIGGER refresh_users_updated_at_step2
  BEFORE UPDATE OF updated_at ON users FOR EACH ROW
  EXECUTE PROCEDURE refresh_updated_at_step2();
CREATE TRIGGER refresh_users_updated_at_step3
  BEFORE UPDATE ON users FOR EACH ROW
  EXECUTE PROCEDURE refresh_updated_at_step3();
```

一見無駄なことをしているように見えるが，これにより

- (A) UPDATE 文で `updated_at` が省略された
- (B) UPDATE 文で `updated_at` に現在と同じ値が渡された

が明確に区別できるようになっている。 `refresh_users_updated_at_step2` が `BEFORE UPDATE OF updated_at` により， (B) の場合にしか実行されないのがポイント。

:::details UPDATE 文で updated_at が省略された
- `refresh_updated_at_step1`
  - `NEW.updated_at = OLD.updated_at` は真であるため， `NEW.updated_at := NULL` が実行される
- `refresh_updated_at_step3`
  - `NEW.updated_at IS NULL` は真であるため， `NEW.updated_at := CURRENT_TIMESTAMP` が実行される
:::

:::details UPDATE 文で updated_at に現在と同じ値が渡された
- `refresh_updated_at_step1`
    - `NEW.updated_at = OLD.updated_at` は真であるため， `NEW.updated_at := NULL` が実行される
- `refresh_updated_at_step2`
    - `NEW.updated_at IS NULL` は真であるため， `NEW.updated_at := OLD.updated_at` が実行される
- `refresh_updated_at_step3`
    - `NEW.updated_at IS NULL` は偽であるため， `NEW.updated_at` は `OLD.updated_at` のままである
:::

:::details UPDATE 文で updated_at に現在と異なる値が渡された
- `refresh_updated_at_step1`
    - `NEW.updated_at = OLD.updated_at` は偽であるため， `NEW.updated_at` は渡された値のままである
- `refresh_updated_at_step2`
    - `NEW.updated_at IS NULL` は偽であるため， `NEW.updated_at` は渡された値のままである
- `refresh_updated_at_step3`
    - `NEW.updated_at IS NULL` は偽であるため， `NEW.updated_at` は渡された値のままである
:::

これらの結果により， MySQL の `ON UPDATE CURRENT_TIMESTAMP` の挙動を正しく再現できていることが分かる。

# まとめ

- `id`
  - Postgres なら `UUID` ネイティブ型の恩恵を受けつつ **UUID v4** を使うと良い。
  - MySQL はシーケンシャルでなければならないので，もし UUID を使いたい場合は  `CHAR(36)` か `BINARY(16)` で **UUID v7** を使う。 `VIRTUAL` な生成カラムを併用するとベター。中央集権で簡易的に済ませたければ `AUTO_INCREMENT` でもアリ。
  - UUID v1 は Docker と相性が悪いので，クライアント側でも採番するなら基本避けたほうが無難。
- `created_at` `updated_at` 
  - Postgres なら **`TIMESTAMPTZ`** 1択。
  - MySQL は基本は **`DATETIME`**，場合によっては数値型で明示的に UTC 基準のタイムスタンプとして対応するのもアリ。後者の場合は `VIRTUAL` のあ生成カラムを併用するとベター。いずれの場合も，精度をマイクロ秒まで引き上げておくと良い。
- `updated_at` の自動更新
  - MySQL は `ON UPDATE CURRENT_TIMESTAMP` で OK。
  - Postgres は頑張ってトリガーを 3 つ書く。

## 付録: そのまま使えるモデルケース

### 簡単に済ませたい

:::details Postgres
```sql
CREATE TABLE users(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP -- 更新はプログラム側で
);
```
:::

:::details MySQL
```sql
CREATE TABLE users(
    id BIGINT PRIMARY KEY UNSIGNED AUTO_INCREMENT,
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6)
);
```
:::

### 正しさを求めたい

:::details Postgres
関数定義は省略しています。
```sql
CREATE TABLE users(
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TRIGGER refresh_users_updated_at_step1
    BEFORE UPDATE ON users FOR EACH ROW
    EXECUTE PROCEDURE refresh_updated_at_step1();
CREATE TRIGGER refresh_users_updated_at_step2
    BEFORE UPDATE OF updated_at ON users FOR EACH ROW
    EXECUTE PROCEDURE refresh_updated_at_step2();
CREATE TRIGGER refresh_users_updated_at_step3
    BEFORE UPDATE ON users FOR EACH ROW
    EXECUTE PROCEDURE refresh_updated_at_step3();
```
:::

:::details MySQL
MySQL は完全に模範解答と言えるものを用意するのが極めて困難です。もう一度よく全体を読んで，トレードオフとしてどこを捨てるかを検討した上で割り切った設計を行ってください。
:::
