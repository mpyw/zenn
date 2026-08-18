---
title: "静的 SQL ジェネレータはなぜ Oracle と相性が悪いのか"
emoji: "😢"
type: "tech"
topics: ["database", "sql", "orm", "oracle", "postgresql"]
published: true
publication_name: "yumemi_inc"
---

:::message
主に Oracle がターゲットの記事ですが，**「なぜ PostgreSQL や MySQL では問題にならないのか」を実測付きで掘り下げています**。 Oracle ユーザ以外の方も是非ご一読ください。
:::

# はじめに

ある案件で，**静的 SQL ジェネレータ**（SQL ファイルを書くと型付きのアクセサコードを生成してくれるタイプのツール）を採用していたバックエンドを，**SQL テンプレート方式**に載せ替えました。きっかけは

**「条件分岐の多い検索クエリが，どう頑張っても Oracle でまともな実行計画にならない」**

という， EXPLAIN を適用して初めて顕在化した性能問題です。 調べていくうちに分かったのは，これが特定のライブラリの出来不出来ではなく，**「アプリケーションから SQL を発行する方式をどう選ぶか」という，言語にもフレームワークにも依存しない設計判断**だったということでした。そして，**同じ問題は他の RDBMS でも起こりうるのに，逃げ道が一切無いのは Oracle だけ**でした。PostgreSQL や MySQL だけを触ってきた人には事前に気づきようがありません。

そこでこの記事では，特定の言語・ライブラリの話に閉じないように， SQL を発行する方式を整理します。 **オプティマイザとの相性** を主眼に置き，各方式の長所・短所を比較します。最後に，方式を変えたときに失われがちな **静的安全性をどう取り戻すか** に触れます。

**対象読者:**

- Oracle ユーザの方
- SQL 実行方式の選定で「結局どれがいいのか」を何度も考えたことがある方
- 実行計画・バインド変数・プリペアドステートメントの挙動を，ざっくりとでも理解している方
- 複数条件の検索画面（いわゆる絞り込み UI）を実装したことがある方

:::message
この記事のコード例は方式の違いを示すためのもので，特定のライブラリの正確な文法とは限りません。実際の記法は各ライブラリのドキュメントを参照してください。
:::

---

# 静的 SQL ジェネレータ方式とは？

まず，この記事の発端になった方式から説明します。

**静的 SQL ジェネレータ**とは，**人間が書いた SQL ファイルをビルド時に解析して，型付きの関数・クラスを生成する**タイプのツールです。

```sql:Go の sqlc の例
-- name: findByDept :many
SELECT emp_no, name, job, dept_no
FROM employees
WHERE dept_no = :deptNo
ORDER BY emp_no;
```

これを食わせると，こういうものが生成されます。

```go:生成されるコード（イメージ）
type FindByDeptParams struct {
    DeptNo int32
}
type FindByDeptRow struct {
    EmpNo  int32
    Name   string
    Job    string
    DeptNo int32
}
func (q *Queries) FindByDept(ctx context.Context, deptNo int32) ([]FindByDeptRow, error) {
    // ...
}
```

代表的な実装は Kotlin の [SQLDelight](https://sqldelight.github.io/sqldelight)，Go の [sqlc](https://sqlc.dev/)，TypeScript の [PgTyped](https://pgtyped.dev/) あたりです。

:::details 言語別の実装一覧

sqlc のエコシステム紹介のようになってしまいますが，**最も多言語に対応しているのは sqlc** で間違いないでしょう。WASM プラグインで出力言語を差し替えられる設計になっており，Go を本体として，Kotlin / Python / TypeScript が公式プラグイン（Beta），それ以外がコミュニティプラグインです。

| 言語 | ツール | sqlc の対応 |
|:---|:---|:---|
| Go | [sqlc](https://sqlc.dev/) | **本体（組み込み）** |
| Kotlin | [SQLDelight](https://sqldelight.github.io/sqldelight) | [sqlc-gen-kotlin](https://github.com/sqlc-dev/sqlc-gen-kotlin)（公式 Beta） |
| TypeScript | [PgTyped](https://pgtyped.dev/)，[Prisma TypedSQL](https://www.prisma.io/typedsql) | [sqlc-gen-typescript](https://github.com/sqlc-dev/sqlc-gen-typescript)（公式 Beta） |
| Python | —— | [sqlc-gen-python](https://github.com/sqlc-dev/sqlc-gen-python)（公式 Beta） |
| Rust | [`sqlx::query!`](https://github.com/launchbadge/sqlx) マクロ<br>（コード生成ではなくコンパイル時検証だが，思想は同系統） | [sqlc-gen-sqlx](https://github.com/mathematic-inc/sqlc-gen-sqlx)（コミュニティ） |
| Java | —— | [sqlc-gen-java](https://github.com/tandemdude/sqlc-gen-java)（コミュニティ） |
| C# | —— | [sqlc-gen-csharp](https://github.com/DaredevilOSS/sqlc-gen-csharp)（コミュニティ） |
| PHP | ——<br>（[phpstan-dba](https://github.com/staabm/phpstan-dba) は検査だけの近縁） | [sqlc-plugin-php-dbal](https://github.com/lcarilla/sqlc-plugin-php-dbal)（コミュニティ） |
| Ruby | —— | [sqlc-gen-ruby](https://github.com/DaredevilOSS/sqlc-gen-ruby)（コミュニティ） |

**「自分の言語には静的 SQL ジェネレータが無い」と思っていても，sqlc 経由なら在るかもしれません。** ただしプラグインの成熟度は本体とは別物なので，採用前に必ず検証してください。
:::

この方式の魅力は圧倒的で，一度体験すると手放したくなくなります。

- **SQL がそのまま見える。** ファイルを開けば発行される SQL が 1 対 1 で書いてある
- **SQL をそのままレビューできる。** DBA やパフォーマンス担当が読める形で PR に載る
- **コンパイル時に検査される。** 列名の誤り・型の不一致・スキーマとの乖離が，実行前に落ちる
- **表現力に上限がない。** ウィンドウ関数でも再帰 CTE でも，SQL で書けるものは何でも書ける

ここまで読むと「じゃあこれでいいのでは」と思えます。実際，**複雑な検索が要件にない単純な CRUD** が中心の領域では問題なく使えます。

問題は，**この方式が「1 クエリ = 1 つの完結した静的な SQL テキスト」という制約と不可分**である点にあります。生成の前提が「SQL テキストが確定していること」なのだから当然です。そしてこの制約が，条件分岐を含むクエリに対して致命的に効いてきます。

その話は評価軸の章で詳しく扱うとして，先に方式を並べましょう。

---

# SQL 実行方式の比較

アプリケーションから SQL を発行する方式は，大きく 4 つに分類できます。

:::message
**ソースコード中への文字列リテラルとしての SQL ベタ書きは除外します。**
:::

## ORM

エンティティ（オブジェクト）を主役に置き，SQL の存在を意識させないことを目指す方式です。

```php:Doctrine ORM の例
// イメージ
$employees = $repository->findBy([
    'deptNo' => 10,
    'job' => 'CLERK',
]);
```

リレーションを辿ればよしなに JOIN や追加クエリが飛び，`save()` すれば差分が UPDATE される。**「SQL を書かない」ことがそのまま価値**になっています。

:::details 言語別のライブラリ

| 言語         | ライブラリ                                                                                                                                                                        |
|:-----------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| PHP        | [**Eloquent**](https://laravel.com/docs/eloquent)（Laravel），[Doctrine ORM](https://www.doctrine-project.org/projects/orm.html)，[Cycle ORM](https://cycle-orm.dev/)             |
| Java       | [Hibernate ORM](https://hibernate.org/orm/)，[Jakarta Persistence](https://jakarta.ee/specifications/persistence/)（仕様），[jOOQ **UpdatableRecord / DAO**](https://www.jooq.org/doc/latest/manual/sql-execution/crud-with-updatablerecords/) |
| Kotlin     | [Exposed **DAO API**](https://www.jetbrains.com/help/exposed/dao-crud-operations.html)，[Komapper **Association API**](https://www.komapper.org/docs/reference/association/)，Hibernate ORM |
| TypeScript | [Prisma](https://www.prisma.io/)，[TypeORM **Repository API**](https://typeorm.io/docs/working-with-entity-manager/working-with-repository/)，[MikroORM](https://mikro-orm.io/)，[Drizzle **Relational Queries**](https://orm.drizzle.team/docs/rqb)，[Objection.js](https://vincit.github.io/objection.js/)，[Sequelize **Model**](https://sequelize.org/docs/v6/core-concepts/model-basics/) |
| Python     | [**ORM**](https://docs.djangoproject.com/en/stable/topics/db/queries/)（Django），[SQLAlchemy **ORM**](https://docs.sqlalchemy.org/en/20/orm/)                                   |
| Ruby       | [**Active Record**](https://guides.rubyonrails.org/active_record_basics.html)（Rails），[Sequel **Model**](https://sequel.jeremyevans.net/rdoc/classes/Sequel/Model.html)        |
| Go         | [GORM](https://gorm.io/)，[Ent](https://entgo.io/)，[bun **Relations**](https://bun.uptrace.dev/guide/relations.html)                                                           |
| Rust       | [SeaORM](https://www.sea-ql.org/SeaORM/)，[Diesel **Associations**](https://diesel.rs/guides/relations.html)                                                                   |
| C#         | [Entity Framework Core](https://learn.microsoft.com/ef/core/)                                                                                                                |
:::

## クエリビルダー

**SQL の構文をそのままメソッドチェーンや DSL に写し取る**方式です。ORM のように SQL を隠すのではなく，SQL を組み立てる過程をプログラミング言語の上に持ってきます。

```php:Laravel の Query Builder の例
// イメージ
$query = $db->table('employees')
    ->select('emp_no', 'name', 'job')
    ->where('dept_no', $deptNo)
    ->orderBy('emp_no');
```

:::details 言語別のライブラリ

| 言語         | ライブラリ                                                                                                                                                                                                      |
|:-----------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| PHP        | [**Query Builder**](https://laravel.com/docs/queries)（Laravel），[Doctrine DBAL **QueryBuilder**](https://www.doctrine-project.org/projects/doctrine-dbal/en/current/reference/query-builder.html)          |
| Java       | [jOOQ](https://www.jooq.org/)，[MyBatis Dynamic SQL](https://mybatis.org/mybatis-dynamic-sql/)，[**Criteria API**](https://jakarta.ee/specifications/persistence/)（Jakarta Persistence）                      |
| Kotlin     | [Exposed **DSL API**](https://www.jetbrains.com/help/exposed/dsl-crud-operations.html)，[Komapper **QueryDsl**](https://www.komapper.org/docs/reference/query/querydsl/)，jOOQ                               |
| TypeScript | [Kysely](https://kysely.dev/)，[Knex](https://knexjs.org/)，[Drizzle **`db.select()`**](https://orm.drizzle.team/docs/select)，[MikroORM **QueryBuilder**](https://mikro-orm.io/docs/query-builder)             |
| Python     | [SQLAlchemy **Core**](https://docs.sqlalchemy.org/en/20/core/)，[PyPika](https://github.com/kayak/pypika)                                                                                                   |
| Ruby       | [**Arel**](https://api.rubyonrails.org/classes/Arel.html)（Rails・Active Record の内部 AST），[Sequel **Dataset**](https://sequel.jeremyevans.net/rdoc/files/doc/dataset_basics_rdoc.html)                        |
| Go         | [bun](https://bun.uptrace.dev/)，[goqu](https://github.com/doug-martin/goqu)，[Squirrel](https://github.com/Masterminds/squirrel)                                                                            |
| Rust       | [Diesel **DSL**](https://diesel.rs/)，[SeaQuery](https://github.com/SeaQL/sea-query)                                                                                                                        |
| C#         | [**LINQ**](https://learn.microsoft.com/ef/core/querying/)（EF Core），[linq2db](https://linq2db.github.io/)，[SqlKata](https://sqlkata.com/)                                                                   |
:::

### ORM とクエリビルダーは，同じライブラリの 2 レイヤーであることが多い

**多くのライブラリは，ORM レイヤーとクエリビルダーレイヤーを両方持っています。** ORM を採用していても，下のレイヤーに降りればクエリビルダーとして使えるということです。

::::details 具体的な対応表と，「ORM レイヤー」の中身の違い

| ライブラリ / フレームワーク | **ORM レイヤー** | **クエリビルダーレイヤー** |
|:---|:---|:---|
| [**Laravel**](https://laravel.com/)（PHP） | [Eloquent](https://laravel.com/docs/eloquent) | [Query Builder](https://laravel.com/docs/queries)（`DB::table()`） |
| [**Doctrine**](https://www.doctrine-project.org/)（PHP） | [Doctrine ORM](https://www.doctrine-project.org/projects/orm.html) | [Doctrine DBAL QueryBuilder](https://www.doctrine-project.org/projects/doctrine-dbal/en/current/reference/query-builder.html) |
| [**Exposed**](https://github.com/JetBrains/Exposed)（Kotlin） | [DAO API](https://www.jetbrains.com/help/exposed/dao-crud-operations.html) | [DSL API](https://www.jetbrains.com/help/exposed/dsl-crud-operations.html) |
| [**Komapper**](https://www.komapper.org/)（Kotlin） | [Association API](https://www.komapper.org/docs/reference/association/) | [QueryDsl](https://www.komapper.org/docs/reference/query/querydsl/) |
| [**jOOQ**](https://www.jooq.org/)（Java） | [UpdatableRecord](https://www.jooq.org/doc/latest/manual/sql-execution/crud-with-updatablerecords/)，[生成 DAO](https://www.jooq.org/doc/latest/manual/sql-execution/daos/) | jOOQ DSL |
| [**SQLAlchemy**](https://www.sqlalchemy.org/)（Python） | [ORM](https://docs.sqlalchemy.org/en/20/orm/)（`Session` / declarative） | [Core](https://docs.sqlalchemy.org/en/20/core/)（`select()` / `Table`） |
| [**Rails**](https://rubyonrails.org/)（Ruby） | [Active Record](https://guides.rubyonrails.org/active_record_basics.html) | [Arel](https://api.rubyonrails.org/classes/Arel.html) |
| [**Sequel**](https://sequel.jeremyevans.net/)（Ruby） | [Model](https://sequel.jeremyevans.net/rdoc/classes/Sequel/Model.html) | [Dataset](https://sequel.jeremyevans.net/rdoc/files/doc/dataset_basics_rdoc.html) |
| [**TypeORM**](https://typeorm.io/)（TypeScript） | [Repository API](https://typeorm.io/docs/working-with-entity-manager/working-with-repository/) | [QueryBuilder](https://typeorm.io/docs/query-builder/select-query-builder/) |
| [**MikroORM**](https://mikro-orm.io/)（TypeScript） | EntityManager | [QueryBuilder](https://mikro-orm.io/docs/query-builder) |
| [**Drizzle**](https://orm.drizzle.team/)（TypeScript） | [Relational Queries](https://orm.drizzle.team/docs/rqb)（`db.query`） | [`db.select()`](https://orm.drizzle.team/docs/select) |
| [**Objection.js**](https://vincit.github.io/objection.js/) + [**Knex**](https://knexjs.org/)（TypeScript） | Objection.js（別パッケージ） | Knex |
| [**bun**](https://bun.uptrace.dev/)（Go） | [Relations](https://bun.uptrace.dev/guide/relations.html)（`Relation()`） | [`NewSelect()`](https://bun.uptrace.dev/guide/query-select.html) |
| [**SeaORM**](https://www.sea-ql.org/SeaORM/)（Rust） | SeaORM 本体 | [SeaQuery](https://github.com/SeaQL/sea-query) |
| [**Diesel**](https://diesel.rs/)（Rust） | [Associations](https://diesel.rs/guides/relations.html) | Diesel DSL |
| [**Kysely**](https://kysely.dev/)（TypeScript） | **——（持たない）** | Kysely 本体 |
| [**goqu**](https://github.com/doug-martin/goqu) / [**Squirrel**](https://github.com/Masterminds/squirrel)（Go） | **——（持たない）** | 本体 |
| [**PyPika**](https://github.com/kayak/pypika)（Python） | **——（持たない）** | 本体 |

:::message
**「ORM レイヤーがある」の中身は，ライブラリによってかなり違います。**

- **フル ORM** …… 変更追跡・遅延ロード・オブジェクトグラフ永続化まで持つ。Hibernate，Doctrine ORM，Eloquent，Active Record など
- **アクティブレコード風の CRUD 補助** …… 主キーを持つ行を `store()` / `delete()` できるが，オブジェクトグラフの永続化はしない。jOOQ の [UpdatableRecord](https://www.jooq.org/doc/latest/manual/sql-execution/crud-with-updatablerecords/) がこれで，作者自身「[jOOQ は full fledged ORM ではない](https://blog.jooq.org/how-to-use-jooqs-updatablerecord-for-crud-to-apply-a-delta/)」と述べています
- **取得済みエンティティの関連ナビゲーション** …… Komapper の [Association API](https://www.komapper.org/docs/reference/association/) は `@KomapperOneToMany` などの注釈から関連を辿る関数を生成しますが，辿る先は `include` で **既に取得済みの `EntityStore`** であって，遅延ロードは発生しません
- **ネストした読み取り専用の関連取得** …… Drizzle の [Relational Queries](https://orm.drizzle.team/docs/rqb) は 1 本の SQL でネストしたオブジェクトを返す読み取り用 API で，コアのビルダーの上に **オプトインで乗る層**です

**「ORM だから SQL が見えなくなる」の度合いは，この差にほぼ比例します。** 下 2 つは，本記事が問題視する「SQL の視認性が落ちる」ケースにはあまり当てはまりません。
:::

:::message
**Doctrine は Symfony 専用のライブラリではありません。** 単体で使えるスタンドアロンのライブラリで，Symfony が標準の永続化層として採用しているにすぎません。そのため本記事では `Doctrine ORM`（フレームワーク名を付けない）と表記しています。
:::
::::

## 静的 SQL ジェネレータ

冒頭の「静的 SQL ジェネレータ方式とは？」で説明したものです。**SQL ファイルが正本で，コードは生成物**という向きになります（例とライブラリ一覧はそちらを参照）。

## SQL テンプレート

**SQL ファイルを書く点は静的 SQL ジェネレータと同じですが，制御構文を埋め込んで動的に組み立てられる**方式です。

:::message
とくに **ディレクティブを SQL コメントとして書き，ファイル単体でも実行できる状態を保つ**ものは **2-way SQL** と呼ばれます。
:::

```sql:Kotlin の Komapper の例 (2-way SQL) 
SELECT emp_no, name, job, dept_no
FROM employees
WHERE 1 = 1
/*%if deptNo != null */
  AND dept_no = /* deptNo */10
/*%end */
/*%if job != null */
  AND job = /* job */'CLERK'
/*%end */
ORDER BY emp_no
```

肝は，**この SQL ファイルをそのままコピーして SQL クライアントに貼ると，何も直さずに実行できる**ことです。`/*%if */` はただのブロックコメントなので無視され，`/* deptNo */10` はコメントの直後に置かれたテスト用リテラル `10` が使われます。実行時にはテンプレートエンジンが `10` をプレースホルダに差し替え，条件が偽なら **`AND dept_no = ?` という行そのものが SQL テキストから消えます**。 「SQL クライアントで動かす向き」と「アプリから実行する向き」の 2 通りに使えるので **2-way** です。

この方式の起源は日本の Seasar プロジェクトの [**S2Dao**](https://www.seasar.org/wiki/?S2Dao) で，そこから [**Doma**](https://docs.domaframework.org/)（Java）に受け継がれました。Doma の作者による Kotlin 版が [**Komapper**](https://www.komapper.org/) です。

---

# SQL 実行方式の長所・短所

次の観点で並べます。

|                             | **ORM**<br>（抽象度：高） | **クエリビルダー**<br>（抽象度：低〜中） | **静的 SQL<br>ジェネレータ** | **SQL テンプレート** |
|:----------------------------|:------------------:|:------------------------:|:--------------------:|:--------------:|
| **再利用性**                    |         ◎          |            ◎             |        **✗**         |  △<br>（機構次第）   |
| **SQL の視認性**                |       **✗**        |          △ 〜 ○           |          ◎           |       ◎        |
| **静的安全性<br>(SQL レイヤー)**     |         ◯          |       △ 〜 ◎<br>（ライブラリ差大）        |        **◎◎**        |       △        |
| **静的安全性<br>(Mapping レイヤー)** |         ◎          |       △ 〜 ◎<br>（ライブラリ差大）        |        **◎◎**        |       △        |
| **条件分岐の柔軟性**                |         ◎          |            ◎             |        **✗**         |       ○        |
| **オプティマイザとの相性**             |         ◎          |            ◎             |        **✗✗**        |       ◎        |
| **表現力の上限**                  |     ライブラリ API      |        ライブラリ API         |         SQL          |      SQL       |

上 5 つの軸をひとつずつ見ていきます（**表現力の上限**は「書けるものが SQL そのものか，ライブラリ API に縛られるか」の違いで，本文では随所で触れます）。

## 再利用性

**「述語（WHERE 句の条件）を部品として切り出し，複数のクエリで共有できるか」** という軸です。 ORM とクエリビルダーは，**述語がプログラミング言語の値になる**ので，ここが極めて強くなります。

```kotlin:Kotlin の Exposed（クエリビルダー）の例
// 述語が Op<Boolean> という「値」なので，関数に切り出して使い回せる
fun activeInDept(deptNo: Int): Op<Boolean> =
    (Employees.retired eq false) and (Employees.deptNo eq deptNo)

// 呼び出し側。一覧用と件数用で同じ述語を共有する
listQuery.andWhere { activeInDept(deptNo) }
countQuery.andWhere { activeInDept(deptNo) }
```

一箇所直せば全部に反映され，**一貫性はコンパイラが保証してくれます**。

一方，**静的 SQL ジェネレータはここが極端に弱い**です。1 クエリ = 1 つの完結した SQL テキストである以上，「WHERE 句のこの部分だけを部品として共有する」という概念が無く，やるとしても文字列を切り貼りするくらいしかありません。結果として何が起きるか。

- ほぼ同じ WHERE 句が複数のクエリに **コピペで散らばる**
- 一箇所直したときに他が追随しているかを **機械的に保証できない**
- そのため「この条件はあのクエリと揃っていること」という **検査用コメントを人力で整備・維持する**運用が発生する

また SQL テンプレート方式が **△** なのは，「断片を共有する機構を持っているかどうかがライブラリによる」ためです。持っていない場合も自前で解決できるので，後の章で具体的に扱います。

## SQL の視認性

**「実際に発行される SQL を，人間がどれだけ直接読めるか」** という軸です。これは性能問題の調査コストと，DBA レビューの可否に直結します。

**静的 SQL ジェネレータと SQL テンプレートは，ここが最強**です。ファイルを開けばそこに SQL が書いてあるのだから当然です。

**ORM は最弱**です。`findBy()` の裏でどんな JOIN が飛んでいるのか，N+1 が起きているのか，遅延ロードがいつ発火するのかは，コードを読んでも推計が難しくなります。ログを見て初めて分かることも多いでしょう。

クエリビルダーは **△ 〜 ○ で，ライブラリによって幅が大きい**軸です。ここが後の「推奨する方式」の分岐点になるので，先に整理しておきます。

- **SQL の構文にほぼ 1 対 1 で対応するもの**（Laravel の Query Builder，Java の jOOQ，Go の uptrace/bun など）は，メソッドチェーンを読めば発行される SQL がほぼそのまま浮かびます。**生 SQL の上位互換**として扱えます
- **型システムに寄せて抽象度を上げたもの**（Exposed / Komapper の DSL など）は，型で守られるぶん SQL からの距離が遠くなります。組み立てコードを読んで SQL を「想像する」作業が必要になります

## 静的安全性

**「実行する前に，どれだけ間違いを検出できるか」** です。ここは 2 つに分けて考える必要があります。

- **SQL レイヤー** …… 発行する SQL 自体が妥当か。構文が壊れていないか，実在する列を参照しているか
- **マッピングレイヤー** …… 返ってきた行をアプリ側の型に載せる部分が妥当か。列の型・NULL 許容・カラム数が合っているか

### SQL レイヤー

ORM とクエリビルダーは，構文レベルでは壊しようがありません。人間が SQL 文字列を書かないので，`SELCT` と打ち間違えることも括弧を閉じ忘れることもない。ここは構造的に保証されます。

問題はその先，「実在する列を参照しているか」です。

- **ORM は ◯。** エンティティのプロパティ経由で列を指すので，タイポはコンパイルエラーになります。ただし DQL / HQL / JPQL のような文字列のクエリ言語を使った瞬間に穴が空き，実行時エラーに落ちます
- **クエリビルダーは △ 〜 ◎ で，ライブラリ差が最も大きい軸です。** Laravel の Query Builder や Knex のように列名を文字列で渡すものは，タイポが実行時まで分かりません。一方 jOOQ・Kysely・Diesel のようにスキーマからメタモデルや型を生成するものは，静的 SQL ジェネレータに迫ります
- **静的 SQL ジェネレータは ◎◎。** ビルド時に実スキーマ（あるいはスキーマ定義ファイル）と SQL を突き合わせるので，「存在しない列名」「型の不一致」「スキーマ変更で壊れたクエリ」がコンパイルエラーになります。他方式の「ライブラリのテーブル定義と整合しているか」より一段踏み込んだ検査です
- **SQL テンプレートは △。** SQL は結局ただの文字列であり，列名の誤りは実行してみるまで分かりません

:::message
**SQL テンプレートの中でも [Doma](https://docs.domaframework.org/) や [Komapper](https://www.komapper.org/docs/reference/query/querydsl/command/) は一段強い**です。アノテーションプロセッサが**コンパイル時にテンプレートを検証**し，ディレクティブ構文（`/*%end */` の欠落など）・未使用パラメータ・パラメータの型やメンバまで見ます（Komapper の `@KomapperCommand` は型メンバー検査まで踏み込みます）。

ただし **実スキーマとの照合まではしません。** 存在しない列を書いてもコンパイルは通ります。
:::

なお SQL 方言レベルの妥当性は，**どの方式でも実行時**です。「この関数はこの DB では使えない」「この構文はこのバージョンでは通らない」は実行してみるまで分かりません。

### マッピングレイヤー

- **ORM は ◎。** エンティティに型が付いているので，マッピングはむしろライブラリの本業です。ただし部分列の射影や JOIN 結果を DTO に落とすような場面では型が緩みがちです
- **クエリビルダーは，やはり △ 〜 ◎。** Kysely のように SELECT した列だけを含む型を推論するものもあれば，連想配列や [`stdClass`](https://www.php.net/manual/ja/class.stdclass.php) をそのまま返すものもあります
- **静的 SQL ジェネレータは ◎◎。** クエリごとに行の型を生成するので，射影しようが JOIN しようが常に正確です。この方式の最大の売りと言っていい
- **SQL テンプレートは △。** 結果を受けるクラスは手で宣言し，列名との対応は実行時に解決されます

**クエリビルダーの 2 つのレイヤーの強さは連動します。** SQL レイヤーが強いライブラリはマッピングも強く，弱いライブラリは両方弱い。どちらも「どこまで型で表現したか」という一つの設計判断から出てくるためです。

:::message
**クエリビルダーの弱さは，方式の宿命ではありません。**

DSL で表現できる範囲を型で守り切れば，**静的 SQL ジェネレータに近いところまで行けます**。代償は，表現力が DSL の設計に閉じ込められることです。

つまりこの軸は，どちらの方式でも **「検査を強くするほど，何かを固定することになる」**。クエリビルダーが固定するのは「書ける SQL の形」，静的 SQL ジェネレータが固定するのは「SQL テキストそのもの」です。

**そして本記事の主題は，後者の固定が引き起こす問題です。**
:::

**SQL テンプレートは両レイヤーとも △**，つまり素直に劣化します。この劣化をどう埋めるかが方式を変えるうえでの最大の宿題で，最後の章で扱います。

## 条件分岐の柔軟性

**「実行時の入力に応じて，WHERE 句や ORDER BY 句の構造そのものを変えられるか」** という軸です。

検索画面を思い浮かべてください。条件が 5 個あって，ユーザーはそのうち任意の組み合わせを指定できる。並び順も 3 種類から選べる。これを実装するとき，方式ごとにこうなります。

ORM・クエリビルダー・SQL テンプレートは，指定された条件だけを組み立てればよいので何も困りません。

```kotlin:クエリビルダーの場合（Kotlin / Exposed）
criteria.name?.let   { query.andWhere { Emp.name   eq it } }
criteria.job?.let    { query.andWhere { Emp.job    eq it } }
criteria.deptNo?.let { query.andWhere { Emp.deptNo eq it } }
```

```sql:SQL テンプレートの場合（2-way SQL）
/*%if name != null */   AND name    = /* name */'SCOTT'  /*%end */
/*%if job != null */    AND job     = /* job */'CLERK'   /*%end */
/*%if deptNo != null */ AND dept_no = /* deptNo */10     /*%end */
```

**静的 SQL ジェネレータだけが，ここで壁に激突します。** SQL テキストは静的に固定されているのだから，「条件を消す」ことができない。素直に書くと，たいていこの形になります。

```sql:静的 SQL ジェネレータに寄せると，こうなりがち（catch-all query）
SELECT * FROM employees
WHERE (:name    IS NULL OR name    = :name)
  AND (:job     IS NULL OR job     = :job)
  AND (:deptNo  IS NULL OR dept_no = :deptNo);
```

「そのパラメータが NULL なら条件を無効化する」というやつです。組み合わせの数だけクエリを書き分ける手もありますが，条件 5 個で 32 本，6 個で 64 本になるので現実的ではありません（複数クエリの事前定義や DB 側のビュー / ファンクションに逃がす道もありますが，いずれも別の複雑さを抱え込みます）。

この書き方には SQL Server 界隈で **catch-all query**（あるいは **kitchen sink query**）という名前が付いており，古典的なアンチパターンとして知られています。

:::details 呼び名の出典
- **catch-all query** —— Gail Shaw が 2009 年に自身のブログで書いたのが初出とされます。[SQLServerCentral に再掲された *Revisiting catch-all queries*](https://www.sqlservercentral.com/blogs/revisiting-catch-all-queries) が読めます
- **kitchen sink query** —— 同じパターンの別名です。Erik Darling は「kitchen sink クエリについては多く書かれてきたが，お気に入りは Aaron Bertrand と Gail Shaw のもの」と[述べています](https://erikdarling.com/the-only-thing-worse-than-optional-parameters/)
- Microsoft 自身もこれを **optional parameter** の問題として認めており，[公式ドキュメントに専用のページ](https://learn.microsoft.com/en-us/sql/relational-databases/performance/optional-parameter-optimization)があります（SQL Server 2025 の Optional Parameter Plan Optimization はこの緩和策。後で詳しく触れます）
- なお本記事で何度か引く Erland Sommarskog の [*Dynamic Search Conditions in T-SQL*](https://www.sommarskog.se/dyn-search.html) は，この分野の正典ですが **「catch-all」「kitchen sink」という語は使っていません**。彼は一貫して "dynamic search conditions" と呼びます
:::

なぜアンチパターンなのかが，次の軸です。

## オプティマイザとの相性

**この記事の中心です。**

### 何が起きるか

さっきの catch-all query をもう一度見てください。

```sql:catch-all query（再掲）
SELECT * FROM employees
WHERE (:name    IS NULL OR name    = :name)
  AND (:job     IS NULL OR job     = :job)
  AND (:deptNo  IS NULL OR dept_no = :deptNo);
```

任意条件が N 個あるこの SQL は，**意味的には 2^N 個の別々のクエリ**です。「名前だけで検索」「名前と部署で検索」「何も指定せず全件」……これらは最適な実行計画が全部違います。名前にインデックスがあるなら 1 つ目はインデックススキャンであるべきだし，最後は全表走査で構いません。

ところが **SQL テキストは 1 個**です。

多くの RDBMS は「SQL テキストをキーにして実行計画をキャッシュする」設計になっています。すると **2^N 通りの意味を持つクエリに，実行計画が 1 個しか作られません。** ではその 1 個はどうなるのか。

:::message alert
**「どの組み合わせでも間違いにならない計画」，つまり全表走査に寄ります。**
:::

`(:name IS NULL OR name = :name)` という形が残っている限り，**素直にはインデックスが使えません**。インデックスで絞れるのは述語が `name = 何か` の形をしているときだけですが，この式は「`:name` が NULL なら全行が該当しうる」という可能性を含んでいるからです。

厳密には，オプティマイザには **OR 展開**という抜け道が 1 つあります（`(:name IS NULL) OR (name = :name)` を，NULL 用の全表走査ブランチと非 NULL 用のインデックスブランチの `UNION ALL` に書き換える手）。ただし成立するのは条件が少ないうちだけで，**任意条件が増えるほどコスト比較で選ばれにくくなり，全表走査へ後退します**（後の Oracle 節で実測とともに扱います）。**多条件検索という catch-all 本来の用途では，全表走査に落ちる**と考えてよいです。

:::message
**「最初に実行された組み合わせに最適化されてしまう」わけではありません。**

パラメータスニッフィング（バインドピーク）の話と混同しやすいところですが，別物です。**catch-all では，最初に何を投げようとアクセスパスは全表走査で確定します。**

実際，後の実測では 1 回目から「name だけ指定」を投げています。それでも Oracle は素直に `TABLE ACCESS FULL` を選びました。「たまたま全件検索が先に来たから」ではありません。

覗いた値が効くのは **カーディナリティ推定**のほうです。行数の見積もりが最初の組み合わせに引きずられるので，結合順序のような「推定に依存する部分」だけが不安定になります。
:::

「オプティマイザが賢ければ `:name IS NULL` が偽のときは分岐を消してくれるのでは？」と思うかもしれません。その通りで，**そこが RDBMS ごとの分かれ目**になります。

### 分かれ目 —— 「その実行専用の計画」を作れるか

必要なのは，たったひとつ。**バインド値を定数として式に埋め込み，そのうえで畳み込む**ことです。前半のバインド値をリテラルとして代入する操作が **パラメータ埋め込み（Parameter Embedding）**，後半の簡約が **定数畳み込み（Constant Folding）** で，効くのはこの 2 つがセットになったときです。

さきほどの catch-all に，`name` だけを指定して投げたとしましょう。バインド値を定数として式に代入してから最適化すると，こうなります。

```sql:定数畳み込みが効くと，catch-all はこうなる
-- 元の述語（:b1 = 'name100'，:b2 と :b3 は NULL）
    (:b1 IS NULL OR name    = :b1)
AND (:b2 IS NULL OR job     = :b2)
AND (:b3 IS NULL OR dept_no = :b3)

-- ① 値を代入する
    ('name100' IS NULL OR name    = 'name100')
AND (NULL      IS NULL OR job     = NULL)
AND (NULL      IS NULL OR dept_no = NULL)

-- ② 評価する
    (FALSE OR name = 'name100')
AND (TRUE  OR ...)
AND (TRUE  OR ...)

-- ③ 畳み込む
    name = 'name100'
```

元の 3 つの OR は跡形もなく消え，**`name = 'name100'` というただの等値述語だけが残りました**。ここまで来れば `name` のインデックスがそのまま使えます。

つまり，静的 SQL ジェネレータが書かざるを得なかった catch-all を，オプティマイザが勝手に **「動的 SQL を書いたのと同じ状態」まで戻してくれる** わけです。

:::message
**「バインド値が見えている」ことと「定数畳み込みができる」ことは別物です。**

どの DB も，バインド値を見ることはします。差がつくのは**見た値をどう使うか**です。使い道は 2 通りあります。

---

**使い道 A：推定にだけ使う**

`:b1` が `'name100'` だと知ったうえで，「この述語は何行くらいヒットしそうか」を見積もる。Oracle の**バインドピーク**，SQL Server の**パラメータスニッフィング**がこれです。

**しかし SQL の式は書き換えません。** 述語は `(:b1 IS NULL OR name = :b1)` のまま計画に残ります。

そして **この形のままでは `name` のインデックスは使えません**（前述のとおり，NULL の可能性を含む OR は `name = 何か` に絞れない）。計画としては，全行を読んで 1 行ずつ判定するしかありません。

---

**使い道 B：リテラルとして扱う**

**バインド値が，最初からその位置にリテラルとして書かれていたかのように扱う。** さきほどの **パラメータ埋め込み → 定数畳み込み**（①②③）がこれです。値が式の一部になるので，述語の形そのものが変わります。SQL Server はこの一連の操作を **Parameter Embedding Optimization** と呼びます（[Paul White の解説](https://www.sql.kiwi/2013/08/parameter-sniffing-embedding-and/)）。

---

A は「何行ヒットするか」の精度を上げるだけで，B は「述語の形」を変える。**インデックスを使えるかどうかを決めるのは B です。**

だから catch-all を救えるのは B だけです。Oracle は A までは行いますが，B に進みません。
:::

#### B は「単一の共有計画」とは両立しない

ここで当然の疑問が出ます。**B のほうが明らかに得なのに，なぜやらない DB があるのか。**

**畳み込んだ計画は，その値でしか正しくないから**です。`(:b1 IS NULL OR name = :b1)` を `name = 'name100'` にした計画は，次に `:b1` が NULL で実行されたときには間違っています。つまり **「1 SQL テキスト = 1 計画」の素朴なキャッシュとは両立しません**。

だから B をやるには，次のどちらかの逃げ道が要ります。

1. **計画を毎回作り直す**<br>（＝ キャッシュを諦める）
2. **Nullable なパラメータの状態ごとに，複数の計画をキャッシュする**<br>（＝ キャッシュを分ける）

この観点で並べると，こうなります。

| RDBMS | B（畳み込み）をどう実現するか                                                                                                                                              |
|:---|:-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **MySQL 8** | **①。** そもそも実行計画をキャッシュせず，EXECUTE のたびに最適化する                                                                                                                    |
| **PostgreSQL** | **①。** Custom Plan を毎回作る（既定で選ばれる。後述）                                                                                                                         |
| **SQL Server 2025** | **②（既定）。** [OPPO](https://learn.microsoft.com/en-us/sql/relational-databases/performance/optional-parameter-optimization) が NULL 状態ごとに複数の計画をキャッシュし，各々を畳む（後述） |
| **SQL Server 〜2022** | **①（明示指定時）。** `OPTION (RECOMPILE)` で「この計画はキャッシュしない」と宣言する                                                                                                     |
| **Oracle** | **①も②も無い**                                                                                                                                                   |
| **SQLite** | ——（そもそも畳み込まない。別事情）                                                                                                                                           |

**Oracle だけが，①の宣言手段も，NULL 状態に基づく②も持ちません。** 共有プールでの計画共有が設計の根幹にあり，そこから降りる口が用意されていないのです。

:::message
**「両立しない」のは “素朴な” キャッシュとだけです。**

SQL Server 2025 の OPPO は②の実例です。1 つの SQL テキストに対し「`@p IS NULL` 用」「`@p IS NOT NULL` 用」の複数の計画（query variant）を作ってキャッシュし，実行時に振り分けます。[公式ドキュメント](https://learn.microsoft.com/en-us/sql/relational-databases/performance/optional-parameter-optimization)いわく，各 variant は "**simplifies predicates based on the actual parameter value ... This constant result folding allows the optimizer to generate execution plans that aren't valid in a single reusable plan**"。まさに「畳み込みを，複数キャッシュで両立させる」機構です。

つまり Oracle に無いのは畳み込む能力ではなく，**①の宣言も②の複数キャッシュも用意されていない**という一点です。
:::

:::message
**「リテラルを直書きすればいいのでは」は，実は筋が通っています。**

SQL テキストに値が入るのでキャッシュキーが値ごとに分かれ，B と再利用が両立するからです（後の実測でも，Oracle はリテラルならきちんとインデックスを使います）。

問題は，**値の種類だけキャッシュエントリが増える**こと。共有プールが溢れ，SQL Injection のリスクも上がります。このトレードオフは後で「構造は動的，値はバインド変数」として整理します。
:::

catch-all への耐性は，これでそのまま決まります。

| RDBMS | 既定で畳み込むか | 対処後 |
|:---|:---|:---:|
| **MySQL 8** | **✅ 常に** | ◎ |
| **PostgreSQL** | **✅ 既定で**（コスト比較で自動選択） | ◎ |
| **SQL Server 2025** | **✅ 既定で**（OPPO） | ◎ |
| **SQL Server 〜2022** | **✗**（`OPTION (RECOMPILE)` で解決） | ◎ |
| **Oracle** | **✗✗ 手段が無い** | **✗✗** |
| **SQLite** | **✗**（畳み込まない・ただし別事情） | ○ |

**既定で壊れるのは Oracle・SQL Server（〜2022）・SQLite の 3 つ**です。ただし SQLite は事実上無害，SQL Server は `OPTION (RECOMPILE)` で確実に直り，2025 では既定で直ります。**逃げ道が一切無いのは Oracle だけ** —— これが本記事のタイトルの意味です。

### 実測: 5 つの RDBMS で確かめた

「本当にそうなのか」を，**5 つすべてで実際に測りました**（SQL Server は 2022 と 2025 の両方）。

#### 検証方法

条件は全部同じです。

- `employees` を **20 万行**用意する
- `name` / `job` / `dept_no` に **それぞれインデックス**を張り，統計を取得する
- **`name` だけを指定した catch-all** を，**バインド変数付き**で投げる

```sql:各 DB に投げたクエリ（プレースホルダの記法は DB ごとに読み替え）
SELECT COUNT(*) FROM employees
WHERE (:b1 IS NULL OR name    = :b1)   -- :b1 = 'name100'
  AND (:b2 IS NULL OR job     = :b2)   -- :b2 = NULL
  AND (:b3 IS NULL OR dept_no = :b3);  -- :b3 = NULL
```

理想は **「`name` のインデックスで 1 件引く」**，つまり `WHERE name = ?` と書いたのと同じ計画です。畳み込みが効けば理想どおりになり，効かなければ全表走査になります。

#### 結果

| RDBMS | catch-all + バインドの結果 | 理想（`WHERE name = ?`） | 差 | 畳み込み |
|:---|:---|:---|---:|:---:|
| **MySQL 8.4** | `Index lookup using ix_name (name='name100')`<br>`Handler_read_rnd_next = 0` | 同じ | **1 倍** | ✅ |
| **PostgreSQL 17** | `Index Scan using ix_name`<br>`Index Cond: (name = 'name100')` | 同じ | **1 倍** | ✅ |
| **SQL Server 2025** | logical reads **6**（OPPO 既定 ON） | logical reads 3 | **2 倍** | ✅ |
| **SQL Server 2022** | logical reads **601** | logical reads 3 | **200 倍** | ❌ |
| **Oracle XE 21c** | `TABLE ACCESS FULL`<br>Buffers **788** | `INDEX RANGE SCAN`<br>Buffers 3 | **263 倍** | ❌ |
| **SQLite 3.43** | `SCAN employees`<br>3.88 ms | 0.016 ms | **243 倍** | ❌ |

- **MySQL・PostgreSQL・SQL Server 2025 は理想どおり。** MySQL と PostgreSQL は `EXPLAIN` からバインド変数が消えて実際の値に置き換わり，`job` と `dept_no` の条件は跡形もありません。SQL Server 2025 は OPPO が既定で効いてインデックスシークになります
- **SQL Server 2022・Oracle・SQLite は全表走査。** ただし SQL Server 2022 は `OPTION (RECOMPILE)` を付ければ理想値に一致します（**同じ製品でも 2025 と 2022 で既定の挙動が分かれる**のが象徴的です）

以下，DB ごとに補足します。

#### MySQL —— 考えることが何も無い

MySQL は伝統的に **実行計画をキャッシュしません**。プリペアドステートメントでもパースツリー等の内部構造はキャッシュしますが，最適化は EXECUTE のたびに走ります。したがって毎回の値で畳み込みが効きます。

「再最適化のコストを毎回払っている」とも言えますが，catch-all の文脈では完全に味方です。**設定も書き方も何も要らず，ただ壊れない**というのは 5 つの中でここだけでした。

#### PostgreSQL —— 両方作って，安いほうを採る

PostgreSQL は測定では理想どおりでした。なぜそうなるのかを見ておきます。PostgreSQL には計画の作り方が 2 通りあります。

| | 作り方 | **B（リテラル扱い）** |
|:---|:---|:---:|
| **Custom Plan** | **実行のたびに**，そのときのパラメータ値を定数に置換してから計画を作る | **やる** |
| **Generic Plan** | **パラメータ値に依存しない**計画を 1 つ作り，以降それを使い回す | **やらない** |

**Custom Plan とは，要するに PostgreSQL 版の Parameter Embedding です。** 呼び名が違うだけで，SQL Server が `OPTION (RECOMPILE)` で解禁するものと同じ。埋め込んだ以上その計画は使い回せないので，「使い捨ての計画」という意味で custom と呼ばれているわけです。

そして PostgreSQL は，どちらか一方を決め打ちしません。**両方を実際に作って，コストを比べて安いほうを採ります。**

1. 最初の **5 回**は Custom Plan で実行し，そのコストの平均を記録する
 （＝ **畳み込んだ計画の平均コスト**）
2. **6 回目**に Generic Plan を 1 つ作り，そのコストを測る
 （＝ **畳み込まなかった計画のコスト**）
3. **安かったほうを以降採用する**

Generic Plan に切り替わること自体は，**問題ではありません**。切り替わるのは「畳み込まないほうが安い」と判断されたとき，つまり畳み込んでも大して得しないときだけだからです。毎回プランニングし直すコストを節約できるぶん，むしろ妥当な判断です。では catch-all ではどちらが勝つのか。考えるまでもありません。

畳み込めば `name = 'name100'` というインデックスの効く述語になり，畳み込まなければ OR が残って全表走査になる。**インデックススキャンが全表走査に負けることはありません。**

実測でも **custom が 9.73，generic が 4334.88** と 445 倍の差がつきました。catch-all は畳み込みの利得が最も大きく出る形なので，そもそも勝負になりません。

そして，これこそが **Oracle との決定的な差**です。

| | 畳み込んだ計画 | 畳み込まない計画 | 選び方 |
|:---|:---:|:---:|:---|
| **PostgreSQL** | 作れる | 作れる | **両方作って安いほうを採る** |
| **Oracle** | **作れない** | 作れる | 選択の余地が無い |

PostgreSQL が安全なのは「たまたま良い方を引いているから」ではなく，**そもそも両方を天秤にかけているから**。Oracle には天秤に載せる片方が存在しません。

#### SQL Server —— 2022 は `OPTION (RECOMPILE)`，2025 は既定（OPPO）

**SQL Server 2022**（互換性レベル 160）は既定では畳み込まれませんが，`OPTION (RECOMPILE)` を付けると理想値に一致します。

| SQL Server 2022 | logical reads |
|:---|---:|
| catch-all + パラメータ（ヒント無し） | 601 |
| catch-all + パラメータ + `OPTION (RECOMPILE)` | **3** |
| `WHERE name = @p1`（理想） | 3 |

そして **SQL Server 2025（互換性レベル 170）では，ヒント無しでも既定で改善します。** OPPO が NULL 状態ごとに複数の計画をキャッシュして振り分けるためです。こちらも実測しました。

| SQL Server 2025 | logical reads |
|:---|---:|
| catch-all + パラメータ（ヒント無し・OPPO 既定 ON） | **6** |
| 同上 + `USE HINT('DISABLE_OPTIONAL_PARAMETER_OPTIMIZATION')`（OPPO 無効） | 601 |
| `WHERE name = @p1`（理想） | 3 |

**ヒント無しで 601 → 6**（20 万行に対しこの reads はインデックスシークが使われた証拠。全表走査なら 601）。OPPO を明示的に切ると 601 に戻るので，差の原因が OPPO であることも確認できました。同じ catch-all が，バージョンが上がるだけで既定で救われるようになったわけです。

#### Oracle —— リテラルなら畳み込めるのに，バインドだとできない

Oracle で同じクエリを **「バインド変数」と「リテラル直書き」の 2 通り** で流すと，こうなりました。

```text:(1) バインド変数 —— 全表走査。3 つの条件がすべて残っている
| Id  | Operation          | Name      | Starts | E-Rows | A-Rows | Buffers |
|   0 | SELECT STATEMENT   |           |      1 |        |      1 |     788 |
|   1 |  SORT AGGREGATE    |           |      1 |      1 |      1 |     788 |
|*  2 |   TABLE ACCESS FULL| EMPLOYEES |      1 |     25 |      1 |     788 |

   2 - filter(((:B2 IS NULL OR "JOB"=:B2) AND (:B3 IS NULL OR "DEPT_NO"=:B3) AND
              (:B1 IS NULL OR "NAME"=:B1)))
```

```text:(2) リテラル直書き —— インデックスが使われ，条件が 1 つに畳み込まれている
| Id  | Operation         | Name    | Starts | E-Rows | A-Rows | Buffers |
|   0 | SELECT STATEMENT  |         |      1 |        |      1 |       3 |
|   1 |  SORT AGGREGATE   |         |      1 |      1 |      1 |       3 |
|*  2 |   INDEX RANGE SCAN| IX_NAME |      1 |      1 |      1 |       3 |

   2 - access("NAME"='name100')
```

同じ表，同じ統計，同じオプティマイザ。違いはリテラルかバインドかだけです。それで Buffers が **788 対 3，263 倍**になりました。

:::message
**これはエディションに依存しません。** 測定は XE ですが，[XE 21c は Enterprise Edition のフル機能セットを同梱](https://blogs.oracle.com/database/oracle-database-21c-xe-generally-available)しており（リソース上限付き），**一番機能が多い状態でも全表走査**でした。しかも catch-all を救えるとしたらそれは述語の畳み込みか OR 展開で，これは全エディション共通のコア CBO の話です。[適応計画のような EE 限定機能](https://mikedietrichde.com/2017/06/12/adaptive-execution-plans-not-available-oracle-se2/)は結合の適応であって述語を畳みません。SE2（適応計画すら無い）でも同じ，むしろ機能が少ないぶん良くなる余地はありません。
:::

:::details Oracle 側で回避できないのか

畳み込めない以上，Oracle に残された道は **「OR を含んだままの述語を，オプティマイザ側の変換でなんとかする」** しかありません。それが **OR 展開**です（12.2 以降のコストベース OR-expansion，11g 以前は CONCATENATION）。**条件が 1 個だけなら，実際にこう展開されてインデックスが使われます**。

```text:条件 1 個なら OR 展開が成立し，非 NULL ブランチでインデックスが使われる
| Id  | Operation                | Name            |
|   2 |   VIEW                   | VW_ORE_xxxxxxxx |
|   3 |    UNION-ALL             |                 |
|   5 |      INDEX FAST FULL SCAN| （PK）           |   ← :name IS NULL のブランチ
|   7 |      INDEX RANGE SCAN    | IX_NAME         |   ← name = :name のブランチ
```

**しかし条件が増えると破綻します。** コストベースの変換なので選ばれないことも多く，任意条件が AND で N 個並ぶと分岐が 2^N に爆発して内部上限に当たります。実測でも，**条件を増やすと OR 展開は消え，`TABLE ACCESS FULL` に落ちました**（本文の測定は 3 条件版）。多条件検索という catch-all 本来の用途では，成立しないと考えてよいです。

手で書く回避策も同じで，**条件が増えると破綻します。**

| 回避策 | 評価 |
|:---|:---|
| `col = NVL(:b, col)` | Oracle が特別扱いしてヒント無しで startup filter 付きの CONCATENATION を生成する。ただし **NOT NULL 列限定**（NULL だと `col = col` が偽になる），かつ **1 〜 2 条件が限界**。`COALESCE` では同じ変換は起きない |
| `UNION ALL` を手書きで展開 | 確実だが条件数に対して指数的に膨らむ。**2 個が限界** |
| `/*+ OR_EXPAND */` ヒント | 条件数が増えると効かなくなる延命策 |
| コメントに乱数を入れて SQL テキストを毎回変える | ハードパースを強制できるが，**共有プールが共有不能カーソルで溢れて library cache mutex 競合を起こす**。インスタンス全体に被害が及ぶので，SQL Server の RECOMPILE より遥かに害が大きい |

なお，畳み込みが効かないと選択率の推定もデフォルト値ベースになります。実務上は **アクセスパスが全表走査に落ちることより，結合順序が崩壊することのほうが致命傷**になりやすいです。

**「他にも手はあるのでは」と思うかもしれませんが，Oracle DBA が挙げがちな候補もすべて外れます。**

- **CURSOR_SHARING** …… リテラルをバインドに置き換える機能。今回とは **逆向き**なので効かない
- **SQL Plan Baselines（SPM）** …… 「この SQL にはこの計画」と固定する仕組み。**NULL かどうかで計画を分けることはできない**
- **Adaptive Cursor Sharing** …… 計画を複数持てるが，「範囲述語」「ヒストグラムのある等値」で発火するもので，**「バインドが NULL か」というスイッチでは反応しない**
- **23ai の新機能**（Real-Time SPM 等）…… いずれも計画の選び方の話で，**バインド値を畳むものではない**

共通するのは，**どれも「バインド値をリテラルとして式に代入する」ところには手が届かない**ことです。
:::

#### SQLite —— そもそも畳み込まない

SQLite だけは事情が違いました。**リテラルを直書きしても `SCAN employees`** のままだったのです。バインド値が見えないから畳み込めないのではなく，**この形の畳み込みを行いません**。

もっとも SQLite は組み込みで，共有プールのようなインスタンス全体の構造を持ちません。**条件ごとに SQL を組み立て直して prepare し直すコストが極めて安い**ので，実務上の困り方は Oracle とはまったく違います。

---

# 依拠すべき原則 —— 「構造は動的，値はバインド変数」

ここまで方式ごとの長所・短所を軸で見てきました。**では，オプティマイザとの相性の問題をどう避けるのか。** その原則を先に据えてから，具体的な方式選定に入ります。

Oracle における多条件検索の古典的な解は，昔から **「構造は動的，値はバインド変数」** と言われてきました。Oracle 社のエンジニアとして Q&A サイト [Ask Tom](https://asktom.oracle.com/) で長年質問に答え続け，Oracle 界隈の常識を形づくってきた **Tom Kyte** 氏の定石です。

Oracle 文化の第一戒律「バインド変数を使え」は，パースが共有プール / library cache という **インスタンス全体で共有される単一構造**を触り，latch / mutex によって **セッション間で直列化する**ことに由来します。しかしこの教えはしばしば **「SQL テキストを 1 個に固定しろ」と誤読されます。** Tom Kyte 氏が言っていたのは「**値**をバインドしろ」であって，「**構造**を固定しろ」ではありません。

ここには **独立した 2 軸**があります。

|                 | **値：埋め込み**                 | **値：バインド変数**                                        |
|:----------------|:---------------------------|:----------------------------------------------------|
| **SQL：静的**      | 不可能                        | **catch-all query**<br>計画が 1 個しか作られず，全パターンが誤った計画を使う |
| **SQL：動的** | 共有プール爆死<br>+ SQL Injection | ⭐ **これが定石**<br>**ここに移りたい**                          |

右下に移ると何が起きるでしょうか。

- 生成される SQL テキストの種類 = **実際に使われる条件の組み合わせ数**。実行回数や値の種類には比例しない
- 検索画面に条件が 10 個あっても，実際に使われる組み合わせは数種類に偏るので，キャッシュされる計画の数は現実的な範囲に収束する
- その一つ一つが **その組み合わせに最適な実行計画**を持ち，何千回も再利用される

catch-all（計画 1 個・全パターンで誤り）と，リテラル直書き（計画が無限）の，**ちょうど良い中間**です。

そして重要なのは，**この性質は特定のライブラリの機能ではない**ということです。

- **ORM も，クエリビルダーも，SQL テンプレートも，すべてこの右下のマスに立っています。**
- **静的 SQL ジェネレータだけが，設計思想として右上のマスに固定されているのです。**

---

# 推奨する方式

ここまでを踏まえた結論です。

## ORM またはクエリビルダー

**但し，抽象度が低いものがその言語にあれば，の話です。**

### 抽象度が低いもの —— 積極的に推せる

[Laravel の Eloquent / Query Builder](https://laravel.com/docs/queries)（PHP），[uptrace/bun](https://bun.uptrace.dev/)（Go）などがここに入ります。

これらは **SQL の視認性と自由度がともに高い**のが特徴です。SQL 文字列を組み立てる過程が見えやすい。メソッドチェーンと発行される SQL がほぼ 1 対 1 に対応するため，「このコードはどんな SQL になるか」を頭の中で追えます。

**実務に即した生 SQL の上位互換**と捉えられます。生 SQL でできることはだいたいでき，そのうえで述語の合成と型による保護が付いてくる。ここを選べる言語にいるなら，これが第一候補です。

### 抽象度が高いもの —— 慎重に

[Doctrine](https://www.doctrine-project.org/projects/orm.html)（PHP），[Exposed](https://github.com/JetBrains/Exposed) / [Komapper](https://www.komapper.org/) の DSL（Kotlin），[Prisma](https://www.prisma.io/)（TypeScript）などがここに入ります。

**抽象度が高くて SQL の視認性が低く**，組み立てコードから発行される SQL を想像する作業が常について回ります。さらに，

- **自由度がライブラリの成熟度に左右される。** 「その構文，このライブラリではまだ書けません」に当たると，そこで詰む
- **冗長な書き方を強いられることがある。** SQL なら 1 行の表現に，DSL では十数行かかることがある

たとえば JetBrains の [Exposed](https://github.com/JetBrains/Exposed) には，執筆時点の 1.4.0 でも **CTE（`WITH` 句）がありません**。4 段の CTE で組み立てられた既存クエリを移そうとすると，導出表（インラインビュー）への書き換えを迫られ，段ごとに付いていた名前が消えて可読性が落ちます。あるいは有志の実装を vendoring することになりますが，そうすると今度は保守対象が増えます。

**「SQL で当たり前に書けるものが DSL では書けない」は，思っているより普通に起こります。** しかもたいてい，移行を決めた後に発覚します。

:::message
**いわゆる「ORM アンチ」と呼ばれる派閥が敵視しているのは，このグループのはずです。**

「ORM は SQL を隠すから悪」という主張は，抽象度の低いクエリビルダーには当てはまりません。議論が噛み合わないときは，たいてい両者が別のものを指しています。
:::

## SQL テンプレート方式

[Doma](https://docs.domaframework.org/)・[MyBatis](https://mybatis.org/mybatis-3/dynamic-sql.html)・[uroboroSQL](https://future-architect.github.io/uroborosql-doc/)（Java），[Komapper のテンプレートクエリ](https://www.komapper.org/docs/reference/query/querydsl/template/)（Kotlin），[go-twowaysql](https://github.com/future-architect/go-twowaysql)（Go），[JinjaSQL](https://github.com/sripathikrishnan/jinjasql)（Python），[HugSQL](https://www.hugsql.org/)（Clojure）などです。

**【2026-08-19 追記】**
著者自身も試験的にライブラリを作ってみました。Go で 2-way SQL を使いたい場合は，こちらも選択肢になります。

[![bisql](https://static.zenn.studio/user-upload/f5aec07f54bb-20260819.png)](https://github.com/mpyw/bisql)

:::message
**専用ライブラリは JVM 系に厚く，それ以外では薄いのが実情です。** 2-way SQL が日本の Seasar 由来という出自と無関係ではないでしょう。

ただし **「専用ライブラリが無いから採れない」わけではありません。** この方式に必要な部品は，突き詰めると次の 2 つだけです。

1. **条件分岐を持つテンプレートエンジン**
2. **値をプレースホルダに逃がし，バインド引数のリストを取り出す仕組み**

1 はどの言語にもあります（Blade，Twig，ERB，Jinja2，`text/template`，……）。2 は展開時に値を配列へ退避してプレースホルダに差し替えるだけで，自前でも大した量になりません。

事実 [JinjaSQL](https://github.com/sripathikrishnan/jinjasql) は **Jinja2 の上に「バインド引数の自動抽出」を薄く乗せただけ**のライブラリです。同じことは Blade でも ERB でもできます。

**専用ライブラリの有無が効いてくるのは，2-way 性（ファイル単体で実行できること）を保ちたい場合**です。汎用テンプレートエンジンのタグは SQL コメントではないので，そこは失われます。
:::

**静的 SQL ジェネレータで問題になった「条件分岐のやりづらさ」と「オプティマイザとの相性」は，この方式でも解決されます。** Komapper の例として `/*%if */` は **条件そのものを SQL テキストから消す**ので，選言が残りません。「構造は動的，値はバインド変数」を満たす手段は，DSL に限らないのです。

そのうえで，**クエリビルダーよりも SQL の視認性がさらに高い**。ファイルを開けばそこに SQL が書いてあり，そのままコピーして SQL クライアントに貼れます。 CTE を何段も使うような複雑なクエリを乱発するアプリケーションで真価を発揮します。

私が実際に載せ替えたときも，クエリビルダー方式とテンプレート方式の **両方を実装して読み比べ**，テンプレート方式を採りました。決め手は視認性です。DSL 方式では「組み立てコードを読んで SQL を想像する」必要があり，既存の SQL からの移植が正しいかを検証するコストが最後まで下がりませんでした。

### TIPS: 断片の再利用性が低いなら，`@include` を自作せよ

前述の比較表で SQL テンプレートの再利用性を **△** としたのは，**断片を共有する機構がライブラリによって弱い**ためです。

:::message
例として挙げた **Komapper には公式の断片共有機構があります。** [`@KomapperPartial` アノテーションと `/*> name */` 記法](https://www.komapper.org/docs/reference/query/querydsl/command/)（COMMAND クエリの機能）で，`null` を渡せば SQL に含まれず，`/*%if */` を含む断片も扱えます。以下の `@include` 自作は，この Partial の制約としての

- Partial は別の Partial を参照できない
- SQL が `.sql` ファイルではなくアノテーション内の文字列になる
- 展開後の SQL を Git 管理できない

を避け， **SQL ファイルのまま再帰展開ありで組み，かつ展開後を Git 管理してレビュー対象としたい** 場合の代替として読んでください。汎用テンプレートエンジンで自作するときの一般論としても使えます。
:::

多くのテンプレート実装は「文字列を差し込む」機構を持っています。しかし **差し込んだ中身が再解析されるとは限りません**。これは実際に踏んだ罠です。Komapper の [Embedded SQL Variables](https://www.komapper.org/docs/reference/query/querydsl/template/)（`/*# name */`）に，`/*%if */` を含む断片を渡すとこうなりました。

```sql:Komapper の Embedded SQL Variables が再解析されない例
-- /*# fragment */ に "/*%if flag */ AND 1 = 1 /*%end */" を渡した結果
SELECT emp_no FROM employees WHERE 1 = 1 /*%if flag */ AND 1 = 1 /*%end */
```

ディレクティブがただの SQL コメントとして残り，`AND 1 = 1` が `flag` の値に関係なく常に効きます。**エラーにならず静かに間違う**ので最悪です。つまりこの機構では `/*%if */` やバインドを含む断片は共有できず，共有したい塊はたいていそれを含みます。

**ここで諦めずに，`@include` ディレクティブを自前で実装してください。**
（今の時代， AI に実装させれば一瞬で終わるでしょう）

```sql:自前の @include を使ったメタテンプレート
WITH visible_depts AS (
/*%! @include fragment/visible-depts.sql */
), target_employees AS (
/*%! @include fragment/target-employees-head.sql */
    AND e.status IN ('0', '2')
/*%! @include fragment/target-employees-filters.sql */
)
SELECT ...
```

`@include` の行はブロックコメントなので，**展開前のファイルをそのまま SQL クライアントに貼っても構文エラーにはなりません**。

**この手のディレクティブは，インクルードだけなら実装のハードルが極めて低いです。** やることは「ファイルを読んで，該当行を参照先の中身で再帰的に置換する」だけ。循環参照の検出と参照先欠落のエラーを入れても 100 行程度で済み，ライブラリ本体に手を入れる必要もありません。

設計のコツは 3 つです。

- **記法をテンプレートエンジンのコメント構文に合わせる。** Komapper のパーサーコメントが `/*%! */` なので，それに寄せました。ディレクティブ族として自然に読めるうえ，ブロックコメントなので断片ファイルも展開先もそのまま SQL クライアントに貼れます（2-way 性を壊さない）
- **展開は実行時ではなくビルド時にやる。** 静的な DRY の仕組みを実行時に解決する理由がありません。ビルド時なら参照先の欠落・循環がビルドで落ちます
- **展開後の SQL も Git 管理し，レビュー対象にする。** 展開後は条件を全部含んだ素の SQL になるので，そのまま `EXPLAIN` にかけられます。生成し忘れは CI で検出できるようにしておきましょう

ライブラリ標準の埋め込み機構がある場合でも，**併用せず `@include` に一本化する**ことをおすすめします。前述のとおり挙動が違ううえ，2 つの機構が並ぶと「どちらで書くべきか」が毎回判断事項になります。

:::message
`@include` の話は Komapper 固有の事情から始まっていますが，**「テンプレートエンジンの埋め込み機構が期待どおり再解析してくれない」という状況自体は方式共通で起こりえます。** 遭遇したら，ライブラリの機能を待つより自分でビルド時展開を書いたほうが早い，というのがここでの教訓です。
:::

---

# 静的安全性はどう担保するか？

静的 SQL ジェネレータを捨てると，**「コンパイル時に SQL が検査される」という最大の長所を失います**。ここを埋めないと，方式変更は単なるトレードオフの付け替えで終わってしまいます。

結論から言うと，**クエリスナップショットを Git 管理してレビュー対象にする**ことで代替できます。やり方次第では，移行前より強い保証すら得られます。

## クエリスナップショット

ほぼすべての方式・ライブラリで，**実行せずに SQL 文字列を取り出せます**（ORM でもクエリビルダーでもテンプレートでも同じ）。これを使い，条件の組み合わせごとに SQL を吐き出し，スナップショットファイルとしてコミットします。パターンごとに 1 ファイルにするのがおすすめです。

ここで大事なのが，**プレースホルダ形（`?`）ではなく，値を埋めた形で書き出す**ことです。そして **テスト入力値だけをヘッダコメントに列挙**します。

```sql:employee-search-by-name.snapshot.sql
-- テスト入力値。以下に無い値はクエリが持つ固定値。
--   name = 'name100'
SELECT emp_no, name, job, dept_no FROM employees
WHERE name = 'name100'
  AND employment_type IN ('FULL_TIME', 'PART_TIME')
ORDER BY emp_no
```

`name = 'name100'` はヘッダに載っているのでテスト入力だと分かります。一方 **`employment_type IN ('FULL_TIME', 'PART_TIME')` はヘッダに無いので，クエリ自身が持つ固定値**（この一覧が対象とする雇用形態の集合）だと読めます。

:::message
**なぜプレースホルダ形ではダメなのか。**

プレースホルダ形のままだと `employment_type IN (?, ?)` としか出ません。**「いくつの区分を取るか」は分かっても「どの区分か」が消える**ので，`FULL_TIME` と `PART_TIME` だけで意図どおりか（`CONTRACT` を含めなくてよいか）をレビューできません。しかも `?` は入力バインドか固定値バインドかの区別も付きません。

値を埋めておけば，**入力値の由来はヘッダで切り分けられ，クエリが埋め込んだ定数はそのまま目に見える**。この方が検査できる情報が多いのです。
:::

このファイルが真価を発揮するのは，**変更したとき**です。たとえば「退職者を一律で除外する」という要件が来たとしましょう。実装では述語を 1 つ足すだけですが，全パターンのスナップショットにこう出ます。

```diff:「退職者を除外する」を足したときの差分
 SELECT emp_no, name, job, dept_no FROM employees
 WHERE name = 'name100'
   AND employment_type IN ('FULL_TIME', 'PART_TIME')
+  AND retired = 0
 ORDER BY emp_no
```

**全パターンのファイルに同じ 1 行が入ったことが，一目で分かります。** もし 1 パターンだけ差分が出ていなければ，そこが漏れです。**「あのクエリにも同じ条件が入っているか」を人間が目視で確かめる作業が，diff を眺めるだけになる**わけです。

**組み合わせの列挙をどうするか**は現実的な論点です。条件数が小さいうちは冪集合（2^N）を全列挙し，増えたら API が実際に受け付ける組み合わせ + 境界ケースに絞るのが妥当でしょう。

:::message
**この発想には先例があります。** Java の [uroboroSQL](https://future-architect.github.io/uroborosql-doc/) は，2-way SQL の分岐に対する **カバレッジ解析機能**を標準で持っています。「テンプレートの分岐を網羅的に検査する」というアイデアは，方式を選んだ人が必ず行き着く場所だということです。
:::

## `EXPLAIN` を CI に載せる

スナップショットテストを実 DB に接続した状態（Testcontainers 等）で走らせれば，SQL 生成時点で方言の解決を通せます。**方言レベルの妥当性という，失ったはずの検査がここで戻ってきます。**

さらに一歩進めます。

```text:CI に載せる検査
主要な組み合わせの実行計画を検証し，主要テーブルに全表走査が出たら fail させる
```

組み合わせごとに SQL テキストが分かれているので，これは素直に書けます。吐き出した SQL を 1 本ずつ `EXPLAIN` にかけ，禁止したい計画が出たら落とすだけです。クエリスナップショット + `EXPLAIN` アサーションは，失った静的検査を取り戻すだけではなく，「性能リグレッション検知」という新しい選択肢となります。

:::message
**PostgreSQL, MySQL, SQL Server なら，静的 SQL ジェネレータのままでもこの検査は成立します。** 畳み込みが効いて，指定した条件のインデックスが使われるからです。実測でも確認したとおりです。
:::

:::message
**「CI で実行計画を検査する」というアイデアにも先例があります。**

PHP の [**phpstan-dba**](https://github.com/staabm/phpstan-dba) は，静的解析の一環として結果セットの型推論・プレースホルダの不整合検出に加えて，**クエリプラン解析**を行います。インデックスが使われていないクエリ・全表走査・非インデックスの過大な読み取りを**エラーとして報告**します。

「実行計画をレビューではなく機械に見せる」という発想自体は，方式に関係なく成立する良い習慣です。**やっていないなら，方式の載せ替えとは無関係に今日から入れる価値があります。**
:::

## 生成物の Git 管理

テンプレート方式で `@include` を導入した場合，管理する成果物は 3 段になります。**全部 Git 管理することをおすすめします。**

```text:@include を導入した場合の成果物 3 段（JVM 系言語の場合のディレクトリ配置イメージ）
X  src/main/sql/**              手書き・正本（@include を含む）
   ↓ ビルド時展開
Y  src/main/resources/sql/**    展開済み。実行時はこれを読むだけ。EXPLAIN 可能
   ↓ 条件を与えて解決
Z  src/test/snapshots/**        指定ごとに解決した SQL
```

冗長に見えますが，**Y があることで「テンプレートの展開結果」と「条件解決の結果」を分けてレビューできます**。`@include` を差し替えたときの影響は Y の差分に，条件を足したときの影響は Z の差分に出ます。Y の生成し忘れは，CI で「生成してみて差分が無いこと」を検査すれば防げます。

---

# まとめ

## catch-all query の逃げ道が Oracle にだけ無い理由

- **静的 SQL ジェネレータは「1 クエリ = 1 つの静的な SQL テキスト」という制約と不可分**であり，条件分岐を含むクエリでは **catch-all query を書きがち**になる
- **`(:b IS NULL OR col = :b)` という形が残る限り，素直にはインデックスが使えない**（OR 展開という抜け道はあるが，条件が増えると破綻する）
- 救うには **バインド値をリテラルとして扱った計画を作る**必要がある。ただしその計画はその値でしか正しくないので，**「1 SQL テキスト = 1 計画」の素朴なキャッシュとは両立しない**。逃げ道は「毎回作り直す」か「状態ごとに複数の計画をキャッシュする」の 2 択
- **MySQL は常に，PostgreSQL は既定で**畳み込む。SQL Server も `OPTION (RECOMPILE)`（〜2022）や OPPO（2025 既定）で解決できる
- **Oracle にだけ，そのどちらの手段も無い。** 無いのは畳み込む能力ではなく，**素朴なキャッシュから降りる術**のほうである（リテラル直書きすれば Oracle も普通にインデックスを使う）

## どうするか

- **原則は「構造は動的，値はバインド変数」。** 特定ライブラリの機能ではなく，**ORM・クエリビルダー・SQL テンプレートが等しく持つ性質**である
- **推奨は，抽象度の低いクエリビルダーか SQL テンプレート方式。** 専用ライブラリが無い言語でも，テンプレートエンジンとバインド引数の抽出があれば作れる（失うのは 2-way 性だけ）
- **失う静的安全性は，クエリスナップショットを Git 管理して埋める。** さらに **`EXPLAIN` アサーションを CI に載せれば性能リグレッション検知の網も張れる**

# 参考

## catch-all query とオプティマイザ

Erland Sommarskog, *Dynamic Search Conditions in T-SQL* —— catch-all query の正典

https://www.sommarskog.se/dyn-search.html

Tom Kyte, *Expert Oracle Database Architecture* / Ask Tom —— 「構造は動的，値はバインド」

https://asktom.oracle.com/

Oracle Database SQL Tuning Guide —— OR Expansion，Adaptive Cursor Sharing

https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/query-transformations.html

PostgreSQL —— `PREPARE` / `plan_cache_mode`（Generic Plan と Custom Plan）

https://www.postgresql.org/docs/current/sql-prepare.html

https://www.postgresql.org/docs/current/runtime-config-query.html

Richard Yen, *The Hidden Behavior of plan_cache_mode* —— Generic Plan に落ちたら戻らない話

https://richyen.com/postgres/2026/03/30/plan_cache_mode.html

MySQL —— Caching of Prepared Statements and Stored Programs

https://dev.mysql.com/doc/refman/8.4/en/statement-caching.html

## SQL テンプレート / 2-way SQL

https://docs.domaframework.org/

https://www.komapper.org/docs/

https://future-architect.github.io/uroborosql-doc/background/

https://github.com/future-architect/go-twowaysql

https://github.com/sripathikrishnan/jinjasql

https://www.hugsql.org/

## 静的 SQL ジェネレータ / 静的検査

https://cashapp.github.io/sqldelight/

https://sqlc.dev/

https://pgtyped.dev/

https://www.prisma.io/typedsql

https://github.com/launchbadge/sqlx

https://diesel.rs/compare/compare_diesel/

https://github.com/staabm/phpstan-dba
