---
title: "後悔しない日付時刻・タイムゾーン設計と Laravel での実践法"
emoji: "🕰️"
type: "tech"
topics: ["php", "laravel", "datetime", "database", "postgresql"]
published: false
---

# はじめに

最近 𝕏 で， Go のプロジェクトで起こっていたタイムゾーンに起因するトラブルをきっかけとして話を広げていったポストがありましたが，そのとき意外と反響がありました。 その一方で，時間をかけて設計された社内の PHP プロジェクトでは比較的トラブル少なく解決できているので，その知見を共有していこうと思います。

（将来的には Go のプロジェクトにも知見を応用できればいいなと考えています）

# データベースについて

## RDBMS の選定

さて，まず Laravel にフォーカスした話をする前に，データベース上での日付時刻のデータ型の選び方というアプローチからこの記事を書こうと思ったのですが，さらにその前に使用する RDBMS を決めなければなりませんね。 MySQL か Postgres を選ぶ状況が多いと思うので，この 2 つに絞って回答します。

***Postgres を使ってください。***

![postgres](https://github.com/mpyw/zenn/assets/1351893/7a078746-c265-4ba1-baf7-55e2bbb92d3d)

機能的に MySQL にしかできないことは僅かながらありますが，今のところほとんど Postgres のほうに優位性があり，日付時刻・タイムゾーンに関しては圧倒的に Postgres のほうが高機能です。また案件の規模によって採用されたりされなかったりの差はあると思いますが，主にフリーランス向けではない本格的な業務システム開発では，フルマネージドの [Aurora (AWS)](https://aws.amazon.com/jp/rds/aurora/) や [AlloyDB (Google Cloud)](https://cloud.google.com/alloydb?hl=ja) といった選択をする機会が増え， Postgres 固有のローレベルなチューニング・メンテナンスのことは殆ど考えなくていいようになってきていると感じます。これまで MySQL しか経験の無かった方もお気軽に是非一度 Postgres を使ってみて，その利便性を体験してみてください。

以下では， Postgres に絞った話をしていきます。まず，データ型に関する話から見ていきましょう。

## 日付時刻に関連するデータ型と演算子

https://www.postgresql.org/docs/current/datatype-datetime.html

上記のリンクから，代表的な日付時刻関連のデータ型をピックアップしていきましょう。エイリアスのあるものはその表記を使用します。

| データ型          | 説明                                              | 例                        |
|:--------------|:------------------------------------------------|:-------------------------|
| `date`        | 日付                                              | `2023-09-01`             |
| `time`        | 時刻                                              | `23:30:00`               |
| `timetz`      | 入力/出力: 特定タイムゾーンにおける時刻<br>**保持: UTC における時刻**     | `23:30:00+09`            |
| `timestamp`   | 日付時刻                                            | `2023-09-01 23:30:00`    |
| `timestamptz` | 入力/出力: 特定タイムゾーンにおける日付時刻<br>**保持: UTC における日付時刻** | `2023-09-01 23:30:00+09` |

:::message
日付時刻はさまざまな表記に対応しており，よく使われる RFC3339 形式の `2023-09-01T23:30:00+09:00` もパースできます。
:::

また Postgres 固有の機能として範囲型というものがあり，日付時刻に対応するものも用意されています。

https://www.postgresql.org/docs/current/rangetypes.html

| 範囲型         | 元のデータ型        | 例                                                                                |
|:------------|:--------------|:---------------------------------------------------------------------------------|
| `daterange` | `date`        | `[2023-01-01,2024-01-01)`<br>（2023年の範囲を表す）                                       |
| `tsrange`   | `timestamp`   | `[2023-01-01 00:00:00,2023-01-02 00:00:00)`<br>（2023年1月1日の範囲を表す）                 |
| `tstzrange` | `timestamptz` | `[2023-01-01 00:00:00+09,2023-01-02 00:00:00+09)`<br>（日本時間を基準とした2023年1月1日の範囲を表す） |

:::message
閉区間は `[` `]`，開区間は `(` `)` で表現します。
例示したように，下限値は閉区間，上限値は開区間とすることが推奨されます。

**両方閉区間にしてしまうと，境界値の扱いが不明瞭になります。**
:::

範囲型に関連する演算子については以下に列挙されています。

https://www.postgresql.org/docs/current/functions-range.html

| 演算子                   | 意味                         |
|:----------------------|:---------------------------|
| *parent* `@>` *child* | *parent* が *child* を包含するか  |
| *child* `<@` *parent* | *child* が *parent* に包含されるか |
| *A* `&&` *B*          | A と B が共通範囲を持つか            |

範囲演算子については， Laravel のコードを使った活用事例を後に掲載します。

### おまけ: 非標準の `timerange` `timetzrange` 型を作るレシピ

残念ながら Postgres 標準機能とは用意されていないのですが，拡張サブタイプを定義して `time` `timetz` の範囲型を作成することも可能です。以下に参考程度にレシピを載せておきます。

:::details timerange
```sql
CREATE OR REPLACE FUNCTION time_subtype_diff(x time, y time) RETURNS float8 AS $$
    SELECT EXTRACT(EPOCH FROM (x - y))
$$ LANGUAGE sql STRICT IMMUTABLE;

DO $$
    BEGIN
        IF NOT EXISTS (SELECT 1 FROM pg_type WHERE typname = 'timerange') THEN
            CREATE TYPE timerange AS RANGE (
                subtype = time,
                subtype_diff = time_subtype_diff
            );
        END IF;
    END
$$;
```
:::

:::details timetzrange
```sql
CREATE OR REPLACE FUNCTION timetz_subtype_diff(x timetz, y timetz) RETURNS float8 AS $$
    SELECT EXTRACT(
        EPOCH FROM (('1970-01-01'::date + x) - ('1970-01-01'::date + y))
    )
$$ LANGUAGE sql STRICT IMMUTABLE;

DO $$
    BEGIN
        IF NOT EXISTS (SELECT 1 FROM pg_type WHERE typname = 'timetzrange') THEN
            CREATE TYPE timetzrange AS RANGE (
                subtype = timetz,
                subtype_diff = timetz_subtype_diff
            );
        END IF;
    END
$$;
```
:::

## タイムゾーンつきデータ型の正体

このうち最も使用機会の多いであろう，完全な日付時刻を格納する `timestamp` `timestamptz` のデータ型について説明します。

一般的にコンピュータサイエンス分野で「タイムスタンプ」というと， **UTC における** `1970-01-01 00:00:00` からの経過秒数をイメージする人が多いと思いますが， **Postgres の `timestamp` はタイムゾーンを特に指定していません**。即ち，それはデータの使い方次第であり，どのタイムゾーンにおける `1970-01-01 00:00:00` からの経過秒数を指すか分からないということになります。

:::message
Postgres に直接関連しませんが，タイムゾーンなしのタイムスタンプのことを **Instant** と呼ぶことがあります。
:::

一方で `timestamptz` は， UTC における `1970-01-01 00:00:00` からの経過秒数を保持するものです。 **格納時に指定したタイムゾーン情報は保持しておらず，入出力の際にクライアントが設定しているタイムゾーンに合わせて `+09` のようなオフセット部分を調整し，実際の日付時刻部分もそれに合わせる** 機能を有しています。どちらかというと，どんなタイムゾーン指定のクライアントに取得されても，本質的にそれが指す地球上の時刻は変化せずに一意に定まるという観点が重要です。

## 各種変換処理

上記の特性を踏まえた上で，様々なデータをどのように相互変換すればいいのか見ていきます。
（プログラミング言語側では任意の方法を採れるでしょうが， SQL 上だけで解決したいケースはあるでしょう）

### `timestamp` → `timestamptz`

タイムゾーンに依存しないローカルタイムスタンプを，あるタイムゾーン基準のタイムスタンプと見なすにはどうすればいいでしょうか？このために **`AT TIME ZONE`** という演算子が用意されています。

:::message alert
`timestamp` に対して `AT TIME ZONE` を使用すると，返り値は `timestamptz` になります。
:::

```sql
SET SESSION TIMEZONE TO 'UTC';
SELECT
    -- 2023-09-01 00:00:00
    -- → 2023-09-01 00:00:00+00
    '2023-09-01 00:00:00'::timestamp AT TIME ZONE 'UTC',

    -- 2023-09-01 00:00:00
    -- → 2023-09-01 00:00:00+09
    -- → 2023-08-31 15:00:00+00
    '2023-09-01 00:00:00'::timestamp AT TIME ZONE 'Asia/Tokyo';
```

後者はそれがパースされるときには `+09` として扱われますが，表示されるタイミングでセッションのタイムゾーン設定を踏まえて `+00` に修正されています。

### `timestamptz` → `timestamp`

では逆に，あるタイムゾーン基準のタイムスタンプを，タイムゾーンに依存しないローカルタイムスタンプに変換する場合はどうすればいいでしょうか？一般的には `timestamp` へのキャストを使います。

```sql
SET SESSION TIMEZONE TO 'UTC';
SELECT
    -- 2023-09-01 00:00:00+09
    -- → 2023-08-31 15:00:00+00
    -- → 2023-08-31 15:00:00
    '2023-09-01 00:00:00+09'::timestamptz::timestamp;
```

ここでもし，全体のタイムゾーン設定と異なるオフセットを基準にしたい場合はどうすべきでしょうか？何とここでまた **`AT TIME ZONE`** 演算子が出てくるのです。

初見のときびっくりしたんですが， Oracle のような他の RDBMS では `AT LOCAL` として別の表現が用意されていたりする一方で， Postgres は `AT TIME ZONE` をどちらの用途にも使用しているようです。

また変換後はタイムゾーンの情報は失われてしまうので，取り扱いに気をつける必要があります。

:::message alert
`timestamptz` に対して `AT TIME ZONE` を使用すると，返り値は `timestamp` になります。
:::

```sql
SET SESSION TIMEZONE TO 'UTC';
SELECT
    -- 2023-09-01 00:00:00+09
    -- → 2023-09-01 00:00:00
    '2023-09-01 00:00:00+09'::timestamptz AT TIME ZONE 'Asia/Tokyo',

    -- 2023-09-01 00:00:00+09
    -- → 2023-08-31 15:00:00+00
    -- → 2023-08-31 15:00:00
    '2023-09-01 00:00:00+09'::timestamptz AT TIME ZONE 'UTC';
```

後者は同じタイムゾーンを指定しているので，キャストと全く同じ動作になります。

### `timestamp` → `date`<br>`timestamptz` → `date`

`timestamptz::date` は，設定されているタイムゾーンに合わせてから日付部分を取り出すことに注意してください。変換後はタイムゾーンの情報は失われてしまうので，取り扱いに気をつける必要があります。

```sql
SET SESSION TIMEZONE TO 'UTC';
SELECT
    -- 2023-09-01
    '2023-09-01 00:00:00'::timestamp::date,

    -- 2023-09-01 00:00:00+09
    -- → 2023-08-31 15:00:00+00
    -- → 2023-08-31
    '2023-09-01 00:00:00+09'::timestamptz::date;
```

### `timestamp` → `time` <br>`timestamptz` → `time`<br>`timestamptz` → `timetz`

時刻に関しても日付と同様です。`timetz` はタイムゾーンの情報を残す一方， `time` への変換後はタイムゾーンの情報は失われてしまうので，取り扱いに気をつける必要があります。

```sql
SET SESSION TIMEZONE TO 'UTC';
SELECT
    -- 00:00:00
    '2023-09-01 00:00:00'::timestamp::time,

    -- 2023-09-01 00:00:00+09
    -- → 2023-08-31 15:00:00+00
    -- → 15:00:00
    '2023-09-01 00:00:00+09'::timestamptz::time;

    -- 2023-09-01 00:00:00+09
    -- → 2023-08-31 15:00:00+00
    -- → 15:00:00+00
    '2023-09-01 00:00:00+09'::timestamptz::timetz,
```

### `timestamp` → `tsrange`<br>`timestamptz` → `tstzrange`

デフォルトで `[)` の形になります。

```sql
SET SESSION TIMEZONE TO 'UTC';
SELECT
    -- [2023-09-01 00:00:00, 2023-09-02 00:00:00)
    tsrange(
        '2023-09-01 00:00:00'::timestamp,
        '2023-09-02 00:00:00'::timestamp
    ),

    -- [2023-08-31 00:00:00+00, 2023-09-01 15:00:00+00)
    tstzrange(
        '2023-09-01 00:00:00+09'::timestamptz,
        '2023-09-02 00:00:00+09'::timestamptz
    );
```

:::message
`[)` 以外のパターンは以下のように第 3 引数を指定します。

```sql
tstzrange(A, B, '[)')
tstzrange(A, B, '[]')
tstzrange(A, B, '()')
```
:::

### `tsrange` → `timestamp`<br>`tstzrange` → `timestamptz`

`lower` `upper` で両端の値を取り出すことができ， `lower_inc` `upper_inc` で閉区間になっているかどうかを取得できます。

```sql
SET SESSION TIMEZONE TO 'UTC';
SELECT
    -- 2023-09-01 00:00:00, true
    lower('[2023-09-01 00:00:00,2023-09-02 00:00:00)'::tsrange),
    lower_inc('[2023-09-01 00:00:00,2023-09-02 00:00:00)'::tsrange),

    -- 2023-09-02 00:00:00, false
    upper('[2023-09-01 00:00:00,2023-09-02 00:00:00)'::tsrange),
    upper_inc('[2023-09-01 00:00:00,2023-09-02 00:00:00)'::tsrange),

    -- 2023-08-31 15:00:00+00, true
    lower('[2023-09-01 00:00:00+09,2023-09-02 00:00:00+09)'::tstzrange),
    lower_inc('[2023-09-01 00:00:00+09,2023-09-02 00:00:00+09)'::tstzrange),

    -- 2023-09-01 15:00:00+00, false
    upper('[2023-09-01 00:00:00+09,2023-09-02 00:00:00+09)'::tstzrange),
    upper_inc('[2023-09-01 00:00:00+09,2023-09-02 00:00:00+09)'::tstzrange);
```

## 具体例から見るデータ型の選び方

よく使われるデータ型と相互変換処理についてまとめました。ここからは，具体例として考えられる事象を洗い出してみて，それをどのようにデータベースに記録するかを考察しましょう。あくまでこの考え方は目安で，実際にはアプリケーションの詳細な事情によって左右されてくることには留意してください。

1. **イベント開催日時**
   - *あるコンサートの開始日時は `2023年5月10日` `午後6時30分` です。*
       - 特定の瞬間を示しているため，`timestamptz` データ型が適切です。
       - 現地時間さえ考えればよい場合は `timestamp` も候補には入りますが，今 **「このイベントは開催中か？」** ということをユーザ向けに表示したい場合はクライアントのタイムゾーンを考慮しなければならないので， `timestamptz` として保存したほうが有利になると考えられます。
         ```sql
         '2023-05-10 06:30:00+09'::timestamptz
         ```

2. **予約期間**
    - *レストランのオンライン予約を受け付ける期間は `2023年5月1日` から `2023年5月31日` です。*
        - 期間を示しているため， `tstzrange` データ型が適切です。時刻情報を補いましょう。また，可能であれば上限値は開区間にしましょう。
          ```sql
          '[2023-05-01 00:00:00+09,2023-06-01 00:00:00+09)'::tstzrange
          ```
    - *レストランの開店時間は `午前11時`，閉店時間は `午後10時` です。*
        - 開店時間と閉店時間をそれぞれ `timetz` データ型のカラムに保存し，アプリケーション側でロジックを組むのが適切です。または，拡張範囲型の `timetzrange` を作成してそれで対応するのも選択肢に入ります。
        - 現地時間さえ考えればよい場合は `time` も候補に入りますが，今 **「この店舗は営業中か？」** ということをユーザ向けに表示したい場合はクライアントのタイムゾーンを考慮しなければならないので， `timetz` として保存したほうが有利になると考えられます。
          ```sql
          '11:00:00+09'::timetz
          '22:00:00+09'::timetz

          -- 拡張範囲型
          '[11:00:00+09,22:00:00+09)'::timetzrange
          ```

3. **商品の賞味期限**
    - *ある商品の賞味期限は `2023年6月30日` です。*
        - 時刻の情報は不要ですので， `date` データ型が適切です。賞味期限は時刻単位で厳密に計算するものではないので， `date` 以外の選択肢はないと思います。
          ```sql
          '2023-06-30'::date
          ```

4. **定期メンテナンスの間隔**
    - *サーバーの定期メンテナンスは，`午前2時` から `午前4時` までです。*
        - 定期的な間隔を示しているため， `timetz` か `time` データ型，あるいはその拡張範囲型が適切です。
          :::message
          `timetz` と `time` のどちらを選択するか，非常に悩ましいケースです。
          `timetz` を採用した場合は，何を指すのかがより自明になります。
          `time` を採用した場合は，仮にこの事業が海外拠点に移動しても，アプリケーション全体のタイムゾーン設定を操作するだけで，データベースのレコードは操作せずとも対応できるかもしれません。
          :::
          ```sql
          '02:00:00'::time
          '04:00:00'::time

          -- 拡張範囲型
          '[02:00:00,04:00:00)'::timerange
          ```

5. **年齢関連**
   - *ユーザー A の生年月日 (Birthdate) は `1990年1月2日` です。*
       - 生年月日は時刻を持たないので `date` データ型が適切です。
       - 但し，より厳密に「生年月日 + 出生時刻」という形で保存する場合は `timestamp` のほうが適切でしょう。この場合は出生地域のタイムゾーン込みで `timestamptz` としてもいいかもしれません。
         ```sql
         '1990-01-02'::date
         ```
   - *ユーザー A の誕生日 (Birthday) は `1月2日` です。*
       - 誕生日は，特定の時間・特定のタイムゾーンに依存せず，かつ年を排除した概念であるため， 月・日をそれぞれぞれ別の情報として保持するのが適切です。
         ```sql
         1::tinyint
         2::tinyint

         -- または birthdate から生成する
         extract(month from '1990-01-02'::date)
         extract(day from '1990-01-02'::date)
         ```

もっと大雑把に大事なところだけをまとめると，以下のように言えるかもしれません。

- **トランザクションデータには，原則的にタイムゾーンを考慮したデータ型を採用する**
- マスタデータはアプリケーションの設計に大きく左右されるので，一概にこれ！と回答することは難しい

https://zenn.dev/dove/articles/c5672dda4c268e

Web アプリケーションの大部分を占めるのは「発生した過去の事象」 「これから起こることを予約した事象」といったトランザクションデータだと思うので，基本方針としては `timestamptz` `timetz` `tstzrange` を選択するとしてしまってもいいとは思います。

# Laravel について

さて概論的な部分はこれぐらいにして，実際に Laravel で使っていくにはどのような準備が必要かを考えましょう。

## 準備編

### [任意] 範囲型の文法拡張ライブラリを導入

https://belamov.github.io/postgres-range/

https://laravel-news.com/postgres-range-type-support-in-laravel-7

もし範囲型を採用する場合は，是非入れて欲しいライブラリです。詳しくはドキュメントを参照してほしいのですが，これによって以下が提供されます。

- マイグレーションでの `daterange` `tsrange` `tstzrange` `timerange` の利用
  - [マクロ](https://github.com/belamov/postgres-range/blob/master/src/Macros/BluePrintMacros.php) を利用して提供されています。
  - `timerange` は拡張データ型としてライブラリ側で用意してくれるようです。<br>（`timetzrange` のサポートは 2023年9月2日 現在では確認できませんでしたが，技術的には容易に対応できそうです）
    ```php
    Schema::create('テーブル名', function (Blueprint $table) {
        $table->dateRange('カラム名');
        $table->timestampRange('カラム名');
        $table->timestampTzRange('カラム名');
        $table->timeRange('カラム名');
    });
    ```
- Eloquent Model でのキャスト
  - 事前にキャスト用のクラスが用意されているので， `protected $casts` の値に列挙するだけですぐ使えます。
    ```php
    protected $casts = [
        'カラム名' => TimestampTzRangeCast::class,
    ];
    ```
- Eloquent Builder, Query Builder での WHERE 条件
  - `whereRaw` で利用してください。範囲型に関しては明示的なキャストを書く必要があるので注意してください。
    ```php
    $query->whereRaw('範囲カラム名 @> ?', [CarbonImmutable::now()]);
    $query->whereRaw('日付時刻カラム名 <@ ?::tstzrange', [new TimestampTzRange(
        from: '2023-01-01 00:00:00+09',
        to: '2023-01-02 00:00:00+09',
    )]);
    ```
    :::message alert
    [マクロ](https://github.com/belamov/postgres-range/blob/master/src/Macros/QueryBuilderMacros.php) が提供されていますが，プレースホルダを利用しない実装になっているのであまり利用はおすすめしません。
    :::
    :::message alert
    2023年9月2日 現在， `TimestampTzRange` クラスの `from` `to` は，文字列で渡さなければ `Y-m-d H:i:s` でフォーマットされてタイムゾーン情報が失われる仕様（バグ？）になっています。くれぐれも `DateTimeInterface` を直接渡さないようにご注意ください。

    多分バグだと思うので，皆様のコントリビューションをお待ちしております。
    :::

### [必須] Query Builder でフォーマットされる `DateTimeInterface` のフォーマットを修正

上で `TimestampTzRange` のタイムゾーン情報ロスについて触れましたが，なんと Laravel 本体側の Query Builder も同じ仕様になっています。（マジかよ…）

とはいえこちらはさすがに毎回「文字列にして渡して」とは言っていられないので， Grammar クラスの修正を行いましょう。

:::details app/Database/PostgresQueryGrammar.php
```php
<?php

declare(strict_types=1);

namespace App\Database;

use Illuminate\Database\Query\Grammars\PostgresGrammar;
use DateTimeInterface;

class PostgresQueryGrammar extends PostgresGrammar
{
    public function getDateFormat(): string
    {
        return DateTimeInterface::RFC3339;
    }
}
```
:::

:::details app/Database/PostgresConnection.php
```php
<?php

declare(strict_types=1);

namespace App\Database;

use Illuminate\Database\PostgresConnection as BasePostgresConnection;

/**
 * @method withTablePrefix(PostgresQueryGrammar $grammar) PostgresQueryGrammar
 */
class PostgresConnection extends BasePostgresConnection
{
    protected function getDefaultQueryGrammar(): PostgresQueryGrammar
    {
        return $this->withTablePrefix(new PostgresQueryGrammar());
    }
}
```
:::

:::details app/Providers/AppServiceProvider.php
```php
<?php

declare(strict_types=1);

namespace App\Providers;

use App\Database\PostgresConnection;
use Illuminate\Database\Connection;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        Connection::resolverFor(
            'pgsql',
            fn (...$args) => new PostgresConnection(...$args),
        );
    }
}
```
:::

これで， `DateTimeInterface` として渡すものは全て RFC3339 フォーマット，即ちタイムゾーン情報を加味した状態で SQL にバインドされるようになります。

> ```php
> $query->whereRaw('範囲カラム名 @> ?', [CarbonImmutable::now()]);
> ```

### [推奨] データベース接続のタイムゾーンを設定

Postgres サーバ側のデフォルトタイムゾーン設定を UTC から変更してしまうことはおすすめしません。影響範囲を小さくするために，クライアント側で都度タイムゾーンを指定することをおすすめします。

```php
// config/database.php

'pgsql' => [
    'driver' => 'pgsql',
    /* ... */
    'timezone' => 'Asia/Tokyo',
],
```

これは必須ではありません。実際に接続のタイムゾーン設定の影響を Laravel アプリケーションが受ける領域は限定的です。

- クエリにバインドされる日付時刻は，通常は `string` ではなく `DateTimeInterface` として渡すはずです。この場合，上記の修正した `PostgresQueryGrammar` によって常に RFC3339 でフォーマットされるため， Postgres には `timestamptz` として解釈されます。 `timestamptz` のオフセット部分がどのような値になっていてもクエリの実行結果には影響しません。
  :::message alert
  以下のように，タイムゾーン情報を持たない文字列を `timestamptz` であるカラムと比較する場合には影響を受けます。タイムゾーン情報を持たない文字列は `timestamp` として解釈されますが，これが `timestamptz` に変換されるときに接続のタイムゾーン設定を考慮します。
  ```php
  User::query()
      ->where('created_at', '2023-01-01 00:00:00')
      ->firstOrFail();
  ```
  :::
- クエリの実行結果として取得する日付時刻は `timestamptz` 型であれば RFC3339 互換のフォーマットで取得されますが，取得された直後は `string` となっています。 RFC3339 のオフセットがどのような値であっても，実質的な意味は変わりません。これが `protected $casts` の設定によって `CarbonImmutable::parse()` で処理されたりするんですが，その際もとの文字列に含まれていたタイムゾーン情報は保持してくれるので，特に大きな問題はありません。
  :::message
  Web ブラウザが API から返却される RFC3339 文字列を `new Date()` でパースした場合， Web ブラウザのタイムゾーンに合わせた表示になりますので，元の値のオフセットは Web アプリケーションの動作に影響することは殆どないでしょう。
  :::

### [必須] Laravel アプリケーションのタイムゾーンを設定

こちらは特に理由がない限り事業拠点のあるタイムゾーン，具体的には `Asia/Tokyo` などの設定にしておいて問題ないでしょう。 `CarbonImmutable::now()` などで取得するオブジェクトすべてに影響するので，使いやすい値にしておきましょう。

```php
// config/app.php

'timezone' => 'Asia/Tokyo',
```

:::message
ここで設定したタイムゾーンは， Laravel アプリケーション起動時に [`date_default_timezone_set`](https://www.php.net/manual/ja/function.date-default-timezone-set.php) 関数によって PHP プロセス全体でグローバルに設定されます。
:::

### [必須] `DateTimeZone` オブジェクトを DI コンテナに登録

`DateTimeInterface` として取り扱っている `DateTimeImmutable` `CarbonImmutable` などのオブジェクトには，任意のタイムゾーンが設定されている可能性があります。**アプリケーション全体で「このオブジェクトにはどのタイムゾーンが設定されているか」というのを正確に把握しきるのは不可能に近い** ので，タイムゾーンに依存する操作をする際には都度アプリケーションのタイムゾーン情報を取れるほうが望ましいです。 とはいえ

*「都度 `config('app.timezone')` をいろいろな場所に書くのはダサい！」*

と感じる人もいるでしょう。それを解消するための方法の 1 つとして，**サービスプロバイダで `DateTimeZone` オブジェクトを登録しておく** のがおすすめです。

```php
// app/Providers/AppServiceProvider.php

$this->app
    ->when(DateTimeZone::class)
    ->needs('$timezone')
    ->giveConfig('app.timezone');
```

この設定を書いておくと， `DateTimeZone` をコンストラクタインジェクションで受けるだけで `config('app.timezone')` の設定をもとに作られたオブジェクトが入ってきます！非常におすすめの方法です。この方法を徹底していれば，クラスのコンストラクタを見ただけで，タイムゾーンに依存する処理が実行されるのかどうかひと目で判断がつくようになります。

### [推奨] ローカル日付時刻を扱うためのライブラリを導入

データベースの日付時刻型がすべて `timestamptz` `timetz` `tstzrange` で一貫して表現されている場合には任意です。 一方で，`timestamp` `date` `time` `tsrange` `daterange` `timerange` などを使っている場合は必須と断言したいレベルに役立つライブラリです。

https://github.com/brick/date-time

日本語に訳して転記しますが，これらにすべて固有の型が与えられていて混同されることがないのは非常に心強いですね。

> | 項目               | 説明                                     | 例                           |
> |------------------|----------------------------------------|-----------------------------|
> | `DayOfWeek`      | 曜日                                     | 月曜日                         | |
> | `Duration`       | 秒+ナノ秒で表される時間                           |                             |
> | `Instant`        | 秒+ナノ秒で表されるローカルタイムスタンプ                  |                             |
> | `Interval`       | `Instant` に挟まれた期間                      |                             |
> | `LocalDate`      | ローカル日付                                 | `2014-08-31`                |
> | `LocalDateRange` | ローカル日付の範囲                              | `2014-01-01`/`2014-12-31`   |
> | `LocalDateTime`  | ローカル日付時刻                               | `2014-08-31T10:15:30`       |
> | `LocalTime`      | ローカル時刻                                 | `10:15:30`                  |
> | `Month`          | 月                                      | 1月                          |
> | `MonthDay`       | 月日                                     | `--12-31`                   |
> | `Period`         | 年月日の量で表される期間                           | 2年3ヶ月4日                     |
> | `TimeZoneOffset` | オフセットベースのタイムゾーン                        | `+01:00`                    |
> | `TimeZoneRegion` | 地域ベースのタイムゾーン                           | `Europe/London`             |
> | `Year`           | 西暦年                                    |
> | `YearMonth`      | 西暦年と月                                  | `2014-08`                   |
> | `ZonedDateTime`  | タイムゾーンつき日付時刻<br>（ネイティブ `DateTime` と同等） | `2014-08-31T10:15:30+01:00` |

これを Eloquent Model のキャストとして用意しておくと非常に使い勝手がいいと思います。最もよく使われそうな `date` 型を `LocalDate` クラスにマップするキャストクラスを例として紹介します。

:::details app/Database/Eloquent/Casts/LocalDate.php

（Larastan 用の注釈を入れています）

```php
<?php

declare(strict_types=1);

namespace App\Database\Eloquent\Casts;

use Brick\DateTime\LocalDate as BrickLocalDate;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use InvalidArgumentException;

/**
 * @phpstan-implements CastsAttributes<BrickLocalDate, BrickLocalDate>
 */
class LocalDate implements CastsAttributes
{
    public function get($model, string $key, $value, array $attributes): BrickLocalDate
    {
        return BrickLocalDate::parse($value);
    }

    public function set($model, string $key, $value, array $attributes): ?string
    {
        if ($value === null) {
            return null;
        }

        if (!$value instanceof BrickLocalDate) {
            throw new InvalidArgumentException('The given value is not a Brick\DateTime\LocalDate instance.');
        }

        return (string)$value;
    }
}
```
:::

## ハマりポイントと回避方法

実際に準備した内容を活用して解決できる問題のうち，特にハマりやすい場面について紹介したいと思います。

### Carbon のフォーマット処理に注意！

Carbon では様々な出力が可能ですが， **Carbon はタイムゾーンを持ったオブジェクト** であることを強く意識してください。タイムゾーンを含まない出力を行ったら，その時点でタイムゾーンの情報が失われてしまいます。これを踏まえて，出力の前に必ずタイムゾーンを狙ったものに設定しておく必要があります。

```php
class FormatDateTimeAction
{
    public function __construct(
        private readonly DateTimeZone $tz,
    ) {
    }

    public function __invoke(DateTimeInterface $time): string
    {
        // 悪い例
        // echo $time->format('Y-m-d H:i:s');

        // 良い例
        echo CarbonImmutable::instance($time)
            ->setTimeZone($this->tz)
            ->format('Y-m-d H:i:s');
    }
}
```

### Carbon の計算処理に注意！

フォーマット処理はあからさまでしたが，ちょっとした日付時刻操作も全て要警戒です。例えば

*「時刻部分を `00:00:00` にする」*

という非常に単純な操作でさえタイムゾーンの影響を受けます。タイムゾーンを無視して時刻部分を `00:00:00` にしてしまったら，タイムゾーンによって実際に何秒間のズレが発生するのかが変わってしまうからです。

```php
class CalculateStartOfDayAction
{
    public function __construct(
        private readonly DateTimeZone $tz,
    ) {
    }

    public function __invoke(DateTimeInterface $time): DateTimeInterface
    {
        // 悪い例
        // return CarbonImmutable::instance()->startOfDay();

        // 良い例
        echo CarbonImmutable::instance($time)
            ->setTimeZone($this->tz)
            ->startOfDay();
    }
}
```

この他にも様々な操作がタイムゾーンの影響を受けるので，計算操作をする際には細心の注意を払ってください。

### `LocalDate` を `DateTimeInterface` に変換する処理に注意！

最後に， `brick/date-time` の `LocalDate` クラスを `DateTimeInterface` に変換する処理で締めくくろうと思います。ローカル日付で入力段階では受けておいて，どこかのレイヤーで実際のタイムゾーン上の日付時刻を指すように変換したい，という需要は多くあると思います。冒頭の方では SQL 上で解決する方法について説明していますが， Laravel アプリケーション側でも同じようにに解決してみましょう。

```php
class ConvertLocalDateToDateTimeAction
{
    public function __construct(
        private readonly DateTimeZone $tz,
    ) {
    }

    public function __invoke(LocalDate $date): DateTimeInterface
    {
        return $date
            ->atTime(LocalTime::midnight())
            ->atTimeZone(TimeZone::fromNativeDateTimeZone($this->tz))
            ->toNativeDateTimeImmutable();
    }
}
```

良いですね！ローカル日付がタイムゾーン上の日付時刻に変換されるまでの過程が，全て明示的に記載されています。暗黙的に処理するよりもずっとバグが生まれにくい，安全なコードになっていることが伝わるでしょうか？

ここでは `CarbonImmutable` への変換をゴールとしましたが，設計次第でライブラリが提供している `ZonedDateTime` として持ち回るのもアリだと思います。ただそこまですると使いにくさのほうが勝りそうなので，私は必須とまでは考えていません。

# まとめ

- Postgres の日付時刻タイムゾーン関連の機能は非常に強力な解決策を提供してくれます。タイムゾーンを考慮してくれる型を積極的に採用してください。
- Laravel にこの仕組みを導入するには準備が必要ですが，一度整備してしまえ非常に安全なコードが手に入ります。明日からはもうタイムゾーン関連のバグにはもう怯えずに済むように，ぜひとも実践してみてください。言語やフレームワークに依らない応用も考えられると思うので，そういった形で参考にしていただいても嬉しいです。
