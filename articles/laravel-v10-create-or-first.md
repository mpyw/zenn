---
title: "[Laravel] createOrFirst の登場から激変した firstOrCreate, updateOrCreate に迫る！"
emoji: "🧐"
type: "tech"
topics: ["php", "laravel", "eloquent", "database", "postgresql"]
published: true
publication_name: "yumemi_inc"
---

# TL;DR

- `firstOrCreate()` `updateOrCreate()` という機能がもともと Eloquent に備わっていたが，これらはレースコンディションを考慮した実装になっていなかったため，大きなアクセス数が伴うプロダクションで安心して使うには少し工夫が必要な機能だった。
- Laravel **[v10.29.0](https://github.com/laravel/framework/releases/tag/v10.29.0)** で `createOrFirst()` という機能が実装され，さらに `firstOrCreate()` `updateOrCreate()` が内部的にそれを利用するように変更された。

:::message alert
正確には `v10.20.0` あたりから徐々に変更が適用されていきましたが，途中で不具合が見つかったため `v10.25.0` で一度リバートされています。**必ず `v10.29.0` 以降を使用してください。**
:::

# はじめに

ご無沙汰しております。最近記事を書く機会がめっきり減ってしまいましたが，今回はかなり強い動機を伴う出来事があったため書くに至りました。今回は Eloquent まわりの最新事情に関する告知になります。

以前 Qiita に以下のような記事を投稿していました。今回は Zenn に投稿していますが，過去の記事を引用して復習から入ろうと思います。

https://qiita.com/mpyw/items/f92f197d6e7f0df231a1

前回はユニークキー制約を利用したリトライ方式以外にも，明示的に悲観ロック・楽観ロックに相当する処理も含めて紹介していましたが，今回は割愛します。

# Laravel 10.19 までの事情

## 機能の復習

Model, Eloquent Builder, 各種 Relation 上では以下のようなメソッドを利用することができます。

```php
// Eloquent Builder
$user = User::query()->updateOrCreate(
    ['email' => 'example@example.com'],
    ['name' => 'Example'],
);

// Model (Eloquent Builder が結局呼び出されるので実質的に ↑ と等価)
$user = User::updateOrCreate(
    ['email' => 'example@example.com'],
    ['name' => 'Example'],
);

// Relation (ここでは HasMany を想定)
$article = $user
    ->articles()
    ->updateOrCreate(
        ['slug' => 'laravel-v10-create-or-first'],
        ['content' => '...'],
    );
```

| メソッド名                                                                                                                                                         | 動作                                                       |
|:--------------------------------------------------------------------------------------------------------------------------------------------------------------|:---------------------------------------------------------|
| [`firstOrNew`](https://github.com/laravel/framework/blob/b8557e4a708a1bd2bc8229bd53feecfa2ac1c6fb/src/Illuminate/Database/Eloquent/Builder.php#L540-L554)     | SELECT して存在したら返す，<br>無ければ新規インスタンスを返す                     |
| [`firstOrCreate`](https://github.com/laravel/framework/blob/b8557e4a708a1bd2bc8229bd53feecfa2ac1c6fb/src/Illuminate/Database/Eloquent/Builder.php#L556-L572)  | SELECT して存在したら返す，<br>無ければ新規インスタンスを INSERT して返す           |
| [`updateOrCreate`](https://github.com/laravel/framework/blob/b8557e4a708a1bd2bc8229bd53feecfa2ac1c6fb/src/Illuminate/Database/Eloquent/Builder.php#L574-L586) | SELECT して存在したら UPDATE して返す，<br>無ければ新規インスタンスを INSERT して返す |

:::message
どのメソッドも，第1引数が **一意な検索条件 `$attributes`**，第2引数が作成または更新時に追加で埋めたいその他の値 `$values` です。
:::

ところが，これらは全て **レースコンディション** のことが考慮されておらず，複数ユーザが同時に操作を行うと以下のような競合動作が発生するかもしれません。ユニークキー制約がもし `$attributes` に設定されている場合，ユニークキー制約に違反するエラーが発生し， `QueryException` （Laravel による `PDOException` の継承クラス）がスローされてしまいます。

|       他ユーザ       |           あなた            |
|:----------------:|:------------------------:|
| SELECT<br>(結果なし) |                          |
|        ︙         |                          |
|        ︙         |     SELECT<br>(結果なし)     |
|        ︙         |            ︙             |
|  INSERT<br>(成功)  |            ︙             |
|                  |            ︙             |
|                  | **INSERT<br>(エラー！キー重複)** |

:::message
ユニークキー制約を張ることができない状況に関しては，今回の記事では取り扱いません。原則的には張ることを想定していると考えてください。
:::

## レースコンディションへの対処方法

このエラーをハンドリングして適切に対処するために，以下のようにリトライする必要がありました。

|       他ユーザ       |           あなた            |
|:----------------:|:------------------------:|
| SELECT<br>(結果なし) |                          |
|        ︙         |                          |
|        ︙         |     SELECT<br>(結果なし)     |
|        ︙         |            ︙             |
|  INSERT<br>(成功)  |            ︙             |
|                  |            ︙             |
|                  | **INSERT<br>(エラー！キー重複)** |
|                  |          リトライ！           |
|                  |   **SELECT<br>(結果あり)**   |

```php
try {
    $user = User::firstOrCreate(['email' => 'example@example.com']);
} catch (QueryException $e) {
    // 実際には $e でエラーコード判定
    $user = User::firstOrCreate(['email' => 'example@example.com']);
}
```

```php
// https://github.com/mpyw/laravel-retry-on-duplicate-key を利用する場合
$user = DB::retryOnDuplicateKey(
    fn () => User::firstOrCreate(['email' => 'example@example.com']),
);
```

# Laravel 10.29.0 以降ではどうなるか？

## `createOrFirst` および `UniqueConstraintViolationException` の登場

```php
$user = User::createOrFirst(['email' => 'example@example.com']);
```

ここにきて， **[`createOrFirst`](https://github.com/laravel/framework/blob/4989e6de076688ade265e2f1970ab6f0c1b60fcb/src/Illuminate/Database/Eloquent/Builder.php#L573-L587)** という機能が初めて登場しました。このメソッドの実装は以下のようになっています。さり気なく **`UniqueConstraintViolationException`** という例外クラスも登場し，以前は `QueryException` のエラーコードから判定する必要があった部分をフレームワーク側が巻き取ってくれています。

> ```php
> public function createOrFirst(array $attributes = [], array $values = [])
> {
>     try {
>         return $this->withSavepointIfNeeded(fn () => $this->create(array_merge($attributes, $values)));
>     } catch (UniqueConstraintViolationException $e) {
>         return $this->useWritePdo()->where($attributes)->first() ?? throw $e;
>     }
> }
> ```

まだ少しややこしい部分があるので，分解しながら慎重に見ていきましょう。

### 最小のフロー

まずノイズを省いて， `createOrFirst` という名前から連想される最初の実装を書いてみます。実際， [バージョン v10.20.0 時点での初期実装](https://github.com/laravel/framework/blob/a655dca3fbe83897e22adff652b1878ba352d041/src/Illuminate/Database/Eloquent/Builder.php#L573-L587) は本当に以下のようになっていました。

https://github.com/laravel/framework/pull/47973

> ```php
> public function createOrFirst(array $attributes = [], array $values = [])
> {
>     try {
>         return $this->create(array_merge($attributes, $values);
>     } catch (UniqueConstraintViolationException $e) {
>         return $this->where($attributes)->first();
>     }
> }
> ```

|      他ユーザ      |           あなた            |
|:--------------:|:------------------------:|
| INSERT<br>(成功) |                          |
|                | **INSERT<br>(エラー！キー重複)** |
|                |        エラーをキャッチ！         |
|                |   **SELECT<br>(結果あり)**   |

リトライ処理で `firstOrCreate` をラップしたときと少し似ている部分がありますね。ここから完成形に至るまでに，どんな修正が加えられたのかを見ていきましょう。

### Postgres のためにセーブポイントを発行する対応

みなさん，データベーストランザクションに関する **セーブポイント** という機能はご存知でしょうか？…その前にもう1つお聞きしますが，**データベースのトランザクションはネストできない** って知っていましたか？たとえば， Laravel で日頃から平然とこんなコードを書いているかと思います。

```php
// 全体をトランザクションでラップする
DB::transaction(function () {
    // ユーザを取得
    $user = User::query()
        ->where(['email' => 'example@example.com'])
        ->firstOrFail();

    // ユーザのタスクについて反復
    $total = 0;
    $success = 0;
    $error = 0;
    $user->tasks()->each(function (Task $task) use (&$total, &$success, &$error) {
        ++$total;
        try {
            // タスク1件ごとにサブトランザクションでラップする
            DB::transaction(function () use ($task) {
                // タスクについて何かを行う
                $task->doSomething();
                // データベースのレコードを更新
                $task->update(['status' => TaskStatus::COMPLETE]);
                // 通知を送るが，もしここで通信エラーが発生したらサブトランザクションのみロールバックする
                // タスク集合全体の処理は継続する
                $user->notify(new TaskCompleted($task));
            });
            ++$success;
        } catch (NotificationException $e) {
            ++$error;
            Log::warning("Warning: 通知に失敗したのでロールバックしました: {$e->getMessage()}");
        }
    });

    // アクティビティを記録
    $user->activities()->create([
        'action' => 'task_execution',
        'count' => compact('total', 'success', 'error'),
    ]);
});
```

さらっと `DB::transaction()` をネストしてサブトランザクションとか謳っていますが， Laravel はどうやってこれを実現しているのでしょうか？ここで，以下の Postgres のマニュアルを読んでみましょう。

https://www.postgresql.org/docs/current/sql-savepoint.html

> To establish a savepoint and later undo the effects of all commands executed after it was established:
>
>
> ```sql
> BEGIN;
>   INSERT INTO table1 VALUES (1);
>   SAVEPOINT my_savepoint;
>   INSERT INTO table1 VALUES (2);
>   ROLLBACK TO SAVEPOINT my_savepoint;
>   INSERT INTO table1 VALUES (3);
> COMMIT;
> ```
> The above transaction will insert the values 1 and 3, but not 2.

上の説明を踏まえて，トランザクションの内側で発行されるセーブポイントについて分かりやすくまとめると，トランザクションとは以下のような対応関係があることが分かります。

|    | トランザクション   | セーブポイント                           |
|:---|:-----------|:----------------------------------|
| 開始 | `BEGIN`    | `SAVEPOINT <セーブポイント名>`            |
| 確定 | `COMMIT`   | `RELEASE SAVEPOINT <セーブポイント名>`    |
| 取消 | `ROLLBACK` | `ROLLBACK TO SAVEPOINT <セーブポイント名>` |

そして大事なのは以下の 2 点です。

- トランザクションはネストできない。
- セーブポイントはトランザクションの中で発行できるが，これも実際にネストしている訳ではない。 **トランザクション中のある通過点を記録しているだけである。**

特に後者の制約があるせいでそのままでは使いづらく感じるでしょうが， Laravel が **「仮想的なトランザクションのネストレベルは何か」** ということを考慮しながら `DB::transaction()` をただコールするだけでいいように抽象化してくれているのです。

---

…さて，ここまでセーブポイントについて説明しましたが，具体的に何を対応しなければならないでしょうか？実は Postgres には，以下のような重大なルールがあるのです。

:::message alert
**Postgres で一度エラーが発生したトランザクションは，以後 `ROLLBACK TO SAVEPOINT <セーブポイント名>` 以外のどんなクエリを発行しても全てエラーになる。唯一実行可能なこの命令を使ってエラーが発生する前のセーブポイントまでロールバックすれば，トランザクションごとロールバックしなくても回復できる**
:::

https://atsuizo.hatenadiary.jp/entry/2019/02/26/142814

そこで，先ほどちらっと見えた [`withSavepointIfNeeded()`](https://github.com/laravel/framework/blob/4989e6de076688ade265e2f1970ab6f0c1b60fcb/src/Illuminate/Database/Eloquent/Builder.php#L1714-L1727) メソッドが必要という話になってきます。このメソッドによって， **「セーブポイントを発行する必要があるか？」** というのを **「仮想トランザクションのネストレベルが 1 以上か」** という方法で判定してくれているわけです。もし既にトランザクションが存在している場合，更にセーブポイントを作成して，ユニークキー制約エラー発生時にロールバックされる場所をそこまでに限定しています。

```diff
 public function createOrFirst(array $attributes = [], array $values = [])
 {
     try {
-        return $this->create(array_merge($attributes, $values));
+        return $this->withSavepointIfNeeded(fn () => $this->create(array_merge($attributes, $values)));
     } catch (UniqueConstraintViolationException $e) {
         return $this->where($attributes)->first();
     }
 }

+   /**
+    * Execute the given Closure within a transaction savepoint if needed.
+    *
+    * @template TModelValue
+    *
+    * @param  \Closure(): TModelValue  $scope
+    * @return TModelValue
+    */
+   public function withSavepointIfNeeded(Closure $scope): mixed
+   {
+       return $this->getQuery()->getConnection()->transactionLevel() > 0
+           ? $this->getQuery()->getConnection()->transaction($scope)
+           : $scope();
+   }
```

この問題は以下の Issue で私が指摘し，元の `createOrFirst` の作者によって解決されています。

https://github.com/laravel/framework/issues/48143

https://github.com/laravel/framework/pull/48144

### レプリケーション遅延への対応

更に，レプリケーション遅延対策として以下の対応を行いました。

```diff
 public function createOrFirst(array $attributes = [], array $values = [])
 {
     try {
         return $this->withSavepointIfNeeded(fn () => $this->create(array_merge($attributes, $values)));
     } catch (UniqueConstraintViolationException $e) {
-        return $this->where($attributes)->first();
+        return $this->useWritePdo()->where($attributes)->first();
     }
 }
```

`useWritePdo()` とは，データベースの Reader/Writer が分かれているとき，通常 SELECT 系の操作は Reader に行くところを Writer に向けるようにする操作です。最新の状態を取得する必要がない場合は Reader から読み取ればいいのですが，時間がかかったときに以下のようなケースで読み取りに失敗してしまいます。

|          他ユーザ           |                  あなた                  |
|:-----------------------:|:-------------------------------------:|
| Writer に INSERT<br>(成功) |                                       |
|            ︙            |                                       |
|            ︙            |     Writer に INSERT<br>(エラー！キー重複)     |
|            ︙            |               エラーをキャッチ！               |
|            ︙            | **Reader から SELECT<br>(何故か見つからない！？)** |
|            ︙            |                                       |
| **Reader へのレプリケーション到達** |                                       |

しかし， Laravel にはこの問題を解決するための `sticky` というオプションが用意されています。同一リクエスト中に限り，何かデータベース上に書き込みを行ったときは，その後の読み取り操作がすべて Writer に向かうというオプションです。

https://laravel.com/docs/10.x/database#the-sticky-option

> The `sticky` option is an *optional* value that can be used to allow the immediate reading of records that have been written to the database during the current request cycle. If the `sticky` option is enabled and a "write" operation has been performed against the database during the current request cycle, any further "read" operations will use the "write" connection. This ensures that any data written during the request cycle can be immediately read back from the database during that same request.

これが用意されているにも関わらず，なぜ明示的な `useWritePdo()` が必要だったのでしょうか？それは， `sticky` のための「副作用が発生した」という情報が記録されるのは， **クエリ実行が成功した場合に限定されるから** です。

INSERT/UPDATE/DELETE などで内部的に利用される [`Illuminate\Database\Connection::affectingStatement`](https://github.com/laravel/framework/blob/4989e6de076688ade265e2f1970ab6f0c1b60fcb/src/Illuminate/Database/Connection.php#L605-L609) というメソッドがありますが，以下のとおり，クエリ実行結果の `rowCount()` を見て副作用があったかどうかを判定するロジックになっています。そもそも `execute()` で例外が発生してしまったら `recordsHaveBeenModified()` まで到達できませんね。

> ```php
> $statement->execute();
>
> $this->recordsHaveBeenModified(
>     ($count = $statement->rowCount()) > 0
> );
> ```

この問題は私が発見し，以下の PR で修正を行いました。

https://github.com/laravel/framework/pull/48161

### `$attributes` ではなく `$values` によってユニークキー制約エラーが発生した場合の対応

更に， `first()` の結果が得られなかったときのために `?? throw $e` で，フォールバックとしてキャプチャしたユニークキー制約例外を再スローするように修正しました。

```diff
 public function createOrFirst(array $attributes = [], array $values = [])
 {
     try {
         return $this->withSavepointIfNeeded(fn () => $this->create(array_merge($attributes, $values)));
     } catch (UniqueConstraintViolationException $e) {
-        return $this->useWritePdo()->where($attributes)->first();
+        return $this->useWritePdo()->where($attributes)->first() ?? throw $e;
     }
 }
```

これは当初は絶対に発生しないと考えられていたようですが，以下のようなエッジケースで発生することに気づき，急いで対応を行いました。

https://github.com/laravel/framework/issues/48235

https://github.com/laravel/framework/pull/48234

テストコードを引用します。

> ```php
>     public function testCreateOrFirstNonAttributeFieldViolation()
>     {
>         // 'email' and 'screen_name' are unique and independent of each other.
>         EloquentTestUniqueUser::create([
>             'email' => 'taylorotwell+foo@gmail.com',
>             'screen_name' => '@taylorotwell',
>         ]);
>
>         $this->expectException(UniqueConstraintViolationException::class);
>
>         // Although 'email' is expected to be unique and is passed as $attributes,
>         // if the 'screen_name' attribute listed in non-unique $values causes a violation,
>         // a UniqueConstraintViolationException should be thrown.
>         EloquentTestUniqueUser::createOrFirst(
>             ['email' => 'taylorotwell+bar@gmail.com'],
>             [
>                 'screen_name' => '@taylorotwell',
>             ]
>         );
>     }
> ```

このように `email` と `screen_name` がそれぞれ独立してユニークキー制約を持っているとき，

「`email` が重複したら既存レコードを取得してほしいが， **`screen_name` の衝突はエラーとして報告してほしい**」

という気持ちで `createOrFirst()` を実行すると，なんと **何もなかったかのように `NULL` が返ってきてしまう** 仕様になっていました。セマンティクスが大きく変わってしまうため，バグとして認めた上で修正を行いました。

## `firstOrCreate` および `updateOrCreate` の内部動作変更

https://github.com/laravel/framework/pull/47973

https://github.com/laravel/framework/pull/48160

https://github.com/laravel/framework/pull/48192

https://github.com/laravel/framework/pull/48213

https://github.com/laravel/framework/pull/48531

https://github.com/laravel/framework/pull/48533

https://github.com/laravel/framework/pull/48541

https://github.com/laravel/framework/pull/48637

上記の 8 つの PR によって，以下の変更が適用されました。

- `firstOrCreate` は内部で `createOrFirst` を利用
- `updateOrCreate` は内部で `firstOrCreate` を利用

```diff
     public function firstOrCreate(array $attributes = [], array $values = [])
     {
         if (! is_null($instance = $this->where($attributes)->first())) {
             return $instance;
         }

-        return tap($this->newModelInstance(array_merge($attributes, $values)), function ($instance) {
-            $instance->save();
-        });
+        return $this->createOrFirst($attributes, $values);
     }

     public function updateOrCreate(array $attributes, array $values = [])
     {
-        return tap($this->firstOrNew($attributes), function ($instance) use ($values) {
-            $instance->fill($values)->save();
+        return tap($this->firstOrCreate($attributes, $values), function ($instance) use ($values) {
+            if (! $instance->wasRecentlyCreated) {
+                $instance->fill($values)->save();
+            }
         });
     }
```

今後， `firstOrCreate` は以下のような動きをするようになります。
（Reader/Writer の切り替えやセーブポイントのコントロールについては省略しています）

|       他ユーザ       |           あなた            |
|:----------------:|:------------------------:|
| SELECT<br>(結果なし) |                          |
|        ︙         |                          |
|        ︙         |     SELECT<br>(結果なし)     |
|        ︙         |            ︙             |
|  INSERT<br>(成功)  |            ︙             |
|                  |            ︙             |
|                  | **INSERT<br>(エラー！キー重複)** |
|                  |          リトライ！           |
|                  |   **SELECT<br>(結果あり)**   |

…お気づきだと思いますが，これはまさに最初の `firstOrCreate` をリトライで工夫していたときのフローそのものです。以前は

「厳密にやるには [私が書いたライブラリ](https://github.com/mpyw/laravel-retry-on-duplicate-key) に任せたほうがいいよ」

ということで誤魔化していた部分ですが，何と Laravel の `firstOrCreate` 自身が **Reader/Writer の切り替えやセーブポイントのコントロールも含めて**，完全な動きをできるようになりました，やったね！ということで，私が書いたライブラリは役目を終えましたので，リポジトリの方はアーカイブさせてもらいました。今まで使ってくれた方，ありがとうございました。

`updateOrCreate` のほうも同様ですが，こちらは `firstOrCreate` を更に転用しつつ， Laravel が作成したばかりの Eloquent Model の **`$wasRecentlyCreated`** に `true` が設定されることを利用したコードに変更しました。

## `firstOrNew` はどうなるか？

`firstOrNew` は取得だけを行い，無かった場合のレコード作成は行わずに自分で `save` を実行してもらうメソッドですが，これに関しては以前のままです。そのため `firstOrNew` を利用する場合に限り， [私が書いたライブラリ](https://github.com/mpyw/laravel-retry-on-duplicate-key) はまだ以下のように活用する用途が残っていました。

```php
// firstOrCreate っぽい動作をさせる場合
$user = DB::retryOnDuplicateKey(function (
    $user = User::firstOrNew(['email' => 'example@example.com']);
    if (!$user->exists) {
        $user->foo = 'foo';
        $user->bar = 'bar';
        $user->save();
    }
    return $user;
)};
```

```php
// updateOrCreate っぽい動作をさせる場合
$user = DB::retryOnDuplicateKey(function (
    $user = User::firstOrNew(['email' => 'example@example.com']);
    $user->foo = 'foo';
    $user->bar = 'bar';
    $user->save();
    return $user;
)};
```

しかし殆どの場合 `firstOrCreate` `updateOrCreate` でカバーできますし，もし局所的に必要になったとしてもせいぜい以下のようなコードを書いておけば解決します。以前と違って `UniqueConstraintViolationException` をキャッチすればいいだけなので，考えることは少なくて済みますね。ゆえに，ライブラリとしての存在価値が無いに等しいのでアーカイブする方針は変わりません。

```php
// firstOrCreate っぽい動作をさせる場合
$user = User::firstOrNew(['email' => 'example@example.com']);
if (!$user->exists) {
    try {
        // saveOrFail() は save() をトランザクションでラップするメソッド
        // Postgres において外側でトランザクションが外側で張られていることを考慮するなら必須
        $user->foo = 'foo';
        $user->bar = 'bar';
        $user->saveOrFail();
    } catch (UniqueConstraintViolationException) {
        $user = User::query()
            ->useWritePdo()
            ->where(['email' => 'example@example.com'])
            ->firstOrFail();
    }
}
```

```php
// updateOrCreate っぽい動作をさせる場合
$user = User::firstOrNew(['email' => 'example@example.com']);
try {
    $user->foo = 'foo';
    $user->bar = 'bar';
    $user->saveOrFail();
} catch (UniqueConstraintViolationException) {
    $user = User::query()
        ->useWritePdo()
        ->where(['email' => 'example@example.com'])
        ->firstOrFail();
    $user->foo = 'foo';
    $user->bar = 'bar';
    $user->save();
}
```

:::details 【おまけ】 DB::retryOnDuplicateKey() を再現する最小コード

```php
class UniqueConstraint
{
    /**
     * @phpstan-template T
     * @phpstan-param callable(): T $callback
     * @phpstan-return T
     */
    public static function retryOnViolated(callable $callback): mixed
    {
        try {
            return $callback();
        } catch (UniqueConstraintViolationException) {
            // リトライ時は Writer を参照
            DB::recordsHaveBeenModified();
            return $callback();
        }
    }
}
```
:::

# まとめ

> - `firstOrCreate()` `updateOrCreate()` という機能がもともと Eloquent に備わっていたが，これらはレースコンディションを考慮した実装になっていなかったため，大きなアクセス数が伴うプロダクションで安心して使うには少し工夫が必要な機能だった。
> - Laravel **[v10.29.0](https://github.com/laravel/framework/releases/tag/v10.29.0)** で `createOrFirst()` という機能が実装され，さらに `firstOrCreate()` `updateOrCreate()` が内部的にそれを利用するように変更された。
