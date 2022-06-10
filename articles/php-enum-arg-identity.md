---
title: "PHP 8.1 において名前付き引数で NULL と引数省略を区別する方法"
emoji: "🧐"
type: "tech"
topics: ["php", "enum", "null"]
published: true
---

# 問題

以下のようなユーザ情報を格納するテーブルを考える。

```sql
CREATE TABLE users(
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT NOT NULL
);
```

このテーブルに関して，

「指定された ID のユーザの， **指定されたフィールドのみを** 差分更新したい」 

という要求があり，それに合わせて以下のように `UserRepository` クラスを実装した。

```php
class UserRepository
{
    public function update(
        int $id,
        ?string $name = null,
        ?string $description = null,
    ): void {
        // 処理内容はダミー
        $updated = [];
        if ($name !== null) {
            $updated['name'] = $name;
        }
        if ($description !== null) {
            $updated['description'] = $description;
        }
        var_dump([
            "User(ID:$id)'s updated fields" => $updated,
        ]);
    }
}
```

```php
$repository = new UserRepository();
$repository->update(1, 'Bob', "I'm Bob.");
$repository->update(2, name: 'Alice');
$repository->update(3, description: "I'm Tom.");

/*
array(1) {
  ["User(ID:1)'s updated fields"]=>
  array(2) {
    ["name"]=>
    string(3) "Bob"
    ["description"]=>
    string(8) "I'm Bob."
  }
}
array(1) {
  ["User(ID:2)'s updated fields"]=>
  array(1) {
    ["name"]=>
    string(5) "Alice"
  }
}
array(1) {
  ["User(ID:3)'s updated fields"]=>
  array(1) {
    ["description"]=>
    string(8) "I'm Tom."
  }
}
*/
```

テーブルのフィールドが全て `NOT NULL` である場合はこれで問題無かった。
では，以下のように一部のフィールドが `NULL` を取り得る場合，

**「`NULL` と省略したことを区別したい」**

という要望が発生するが，どうすれば良いだろうか？

```sql
CREATE TABLE users(
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT -- ここが NULL 許容になった！
);
```

# 最適解: `enum` でデフォルト引数識別用の型を作ってデフォルト値に充てる

:::message
[UNION 型](https://www.php.net/manual/ja/language.types.declarations.php#language.types.declarations.composite.union) は PHP 8.0 で導入されました。
:::

:::message
[Enum](https://www.php.net/manual/ja/language.types.enumerations.php) は PHP 8.1 で導入されました。
:::

`enum` の本来の使い方としては，マジックナンバー的な使い方をされる `int` や `string` に型安全性をもたらすような用途が挙げられるが，なんとこんな使い方もできる。

```php
enum Arg
{
    case Identity;
}

class UserRepository
{
    public function update(
        int $id,
        string|Arg $name = Arg::Identity,
        null|string|Arg $description = Arg::Identity,
    ): void {
        $updated = [];
        if ($name !== Arg::Identity) {
            $updated['name'] = $name;
        }
        if ($description !== Arg::Identity) {
            $updated['description'] = $description;
        }
        var_dump([
            "User(ID:$id)'s updated fields" => $updated,
        ]);
    }
}
```

### 長所

- 型レベルで完全に識別可能な値を使えている。
- `===` `!==` で素直に比較できる。
- 意図が伝わりやすい。

### 短所

無し

# 別解: `class` でデフォルト引数識別用の型を作ってデフォルト値に `new` で充てる

:::message
[デフォルト引数でのインスタンス生成](https://www.php.net/manual/ja/functions.arguments.php#functions.arguments.default) は， PHP 8.1 で導入されました。
:::

```php
final class Identity
{
}

class UserRepository
{
    public function update(
        int $id,
        string|Identity $name = new Identity(),
        null|string|Identity $description = new Identity(),
    ): void {
        $updated = [];
        if ($name instanceof Identity) {
            $updated['name'] = $name;
        }
        if ($description instanceof Identity) {
            $updated['description'] = $description;
        }
        var_dump([
            "User(ID:$id)'s updated fields" => $updated,
        ]);
    }
}
```

### 長所

- 型レベルで完全に識別可能な値を使えている。

### 短所

- インスタンスが1個1個異なるので， `instanceof` を使わないと比較できない。
- `new` がノイズになって，意図が認識しにくい。
- PHP 8.1 縛りが何れにせよかかるので， `enum` 案に対してのメリットが無い。

:::message alert
```php
define('IDENTITY', new Identity());
```
&nbsp;
としてグローバル定数化する手法を使うと `new` シンタックスが無い PHP 8.0 までの環境でもデフォルト値としてオブジェクトが参照できる…ように思えますが，なんと **`define()` でオブジェクトを受け取るのが PHP 8.1 からの機能** であるため，これも意味がありません。 `enum` の導入と同時にいろいろ対応が入っていたようです…
:::

# おまけ: `func_get_args()` と名前付き引数の関係

```php
enum Arg
{
    case Identity;
}

class UserRepository
{
    public function update(
        int $id,
        string|Arg $name = Arg::Identity,
        null|string|Arg $description = Arg::Identity,
    ): void {
        var_dump(func_get_args());
    }
}
```

```php
$repository = new UserRepository();
$repository->update(1, 'Bob', "I'm Bob.");
$repository->update(2, name: 'Alice');
$repository->update(3, description: "I'm Tom.");

/*
array(3) {
  [0]=>
  int(1)
  [1]=>
  string(3) "Bob" <-- わかる
  [2]=>
  string(8) "I'm Bob." <-- わかる
}
array(2) {
  [0]=>
  int(2)
  [1]=>
  string(5) "Alice" <-- わかる
}
array(3) {
  [0]=>
  int(3)
  [1]=>
  enum(Arg::Identity) <-- えっ，君なんでいるの？
  [2]=>
  string(8) "I'm Tom." <-- わかる
}
*/
```

初見だとちょっと意外な結果に見えるかもしれないが，これも[マニュアル](https://www.php.net/manual/ja/function.func-get-args.php)でしっかり説明されている。

> :::message alert
> 注意: この関数は、渡された引数のみのコピーを返します。 デフォルトの（渡されていない）引数については考慮しません。
> :::

ここまでは PHP 7.4 までの常識。引数が指定された部分までが入ってきて，指定しなかった部分以降は入ってこない。問題は以下の部分。

> :::message alert
> PHP 8.0.0 以降における `func_*()` 関数ファミリは、名前付き引数に関しては、ほぼ透過的に動作するはずです。つまり、 **渡された全ての引数は位置を指定したかのように扱われ、引数が指定されない場合は、デフォルト値で置き換えられるということです**。 この関数は、未知の名前付きの可変長引数を無視します。 未知の名前付き引数は、可変長引数を通じてのみアクセスできます。
> :::

**「名前付き引数が無かった時代だったらこう書く」** というコードに置き換えた上で従来の動きを再現する，という動きになっているようだ。互換性を維持するためには最良の選択だったのだろうか…

# おまけ: PHP 7.4 までの機能で何とかして頑張る

## 絶対に被らない（被らないとは言ってない）値を用意する

:::message alert
[関数パラメータリストの末尾カンマ](https://www.php.net/manual/ja/functions.arguments.php#functions.arguments) は， PHP 8.0 以降でしか使えません。
:::

:::message alert
[名前付き引数](https://www.php.net/manual/ja/functions.arguments.php#functions.named-arguments) は， PHP 8.0 以降でしか使えません。
:::

```php
final class Identity
{
    // 文字列専用。 int 型は？とか言われても知りません
    public const STR = '_________IDENTITY__________';
}

class UserRepository
{
    public function update(
        int $id,
        string $name = Identity::STR,
        ?string $description = Identity::STR
    ): void {
        $updated = [];
        if ($name !== Identity::STR) {
            $updated['name'] = $name;
        }
        if ($description !== Identity::STR) {
            $updated['description'] = $description;
        }
        var_dump([
            "User(ID:$id)'s updated fields" => $updated,
        ]);
    }
}
```

```php
$repository = new UserRepository();
$repository->update(1, 'Bob', "I'm Bob.");
$repository->update(2, 'Alice');
$repository->update(3, Identity::STR, "I'm Tom.");
```

つらい。やめようね。

## 連想配列 + ArrayShape 記法に逃げる

多分 PHP 7.4 までなら一番マシな方法…
PHPStan や PhpStorm で [ArrayShape 記法](https://qiita.com/tadsan/items/bfa9465166c351da37e5) 対応が入っていればどうにかなる。

```php
class UserRepository
{
    /**
     * @param array{name?: string, description?: ?string} $params
     */
    public function update(
        int $id,
        array $params,
    ): void {
        $updated = [];
        if (array_key_exists('name', $params)) {
            $updated['name'] = $params['name'];
        }
        if (array_key_exists('description', $params)) {
            $updated['name'] = $params['description'];
        }
        var_dump([
            "User(ID:$id)'s updated fields" => $updated,
        ]);
    }
}
```

```php
$repository = new UserRepository();
$repository->update(1, ['name' => 'Bob'], ['description' => "I'm Bob."]);
$repository->update(2, ['name' => 'Alice']);
$repository->update(3, ['description' => "I'm Tom."]);
```
