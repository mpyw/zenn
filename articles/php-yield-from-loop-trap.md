---
title: "Generator の yield from を do-while ループの中で使ったら死んだ"
emoji: "☠"
type: "tech"
topics: ["php", "generator", "yield"]
published: true
---

## 問題

`$response = $this->client->getMembers();` において，結果は以下のような応答になるとします。一覧取得系の Web API でよくある形です。

```php
// 1ページ目
/* $response->members()    #=> */ [new Member(id: 1), new Member(id: 2)];
/* $response->nextCursor() #=> */ 'xxx';

// 2ページ目
/* $response->members()    #=> */ [new Member(id: 3), new Member(id: 4)];
/* $response->nextCursor() #=> */ null;
```

このとき，以下のコードは何が問題でしょうか？どこをどう直すべきでしょうか？

```php
/**
 * ページネーションしながら全てのメンバーを遅延処理で取得します。
 * 
 * @return Generator<int, Member> 
 */
public function members(): Generator
{
    // カーソルの初期値は NULL
    $cursor = null;
    do {
        // 現在のカーソルにおけるページを取得
        $response = $this->client->getMembers($cursor);
        // 現在のページのメンバー一覧を返す
        yield from $response->members();
    } while ($cursor = $response->nextCursor()); // 次のカーソルが取れる間は継続
}
```

## 回答

```diff
 /**
  * ページネーションしながら全てのメンバーを遅延処理で取得します。
  * 
  * @return Generator<int, Member> 
  */
 public function members(): Generator
 {
     // カーソルの初期値は NULL
     $cursor = null;
     do {
         // 現在のカーソルにおけるページを取得
         $response = $this->client->getMembers($cursor);
         // 現在のページのメンバー一覧を返す
-        yield from $response->members();
+        foreach ($response->members() as $member) {
+            yield $member;
+        }
     } while ($cursor = $response->nextCursor()); // 次のカーソルが取れる間は継続
 }
```

## 解説

実はPHP 公式マニュアルでも触れられてはいます[^1]。ここではこれを噛み砕き，今回の問題に合わせて説明してみます。

### PHP の `array` は全て言語仕様上は連想配列

他の多くの言語では区別されている部分ですが， PHP ではリストと連想配列をどちらも `array` として扱っています。PHP の最大の特徴の 1 つですね。

:::message
- 言語実装としては区別して最適化されている部分はあります
- `array_is_list()` や `json_encode()` など，「リストとみなせる形になっている配列か」を考慮してリストと連想配列を区別して挙動を変える関数も一部存在します
:::

### `yield from` に配列を与えると連想配列としてのキーも列挙される

```php
yield from [new Member(id: 1), new Member(id: 2)];
yield from [new Member(id: 3), new Member(id: 4)];
```

これを分解するとどうなるでしょうか？こうでしょうか？

```php
yield new Member(id: 1);
yield new Member(id: 2);
yield new Member(id: 3);
yield new Member(id: 4);
```

…いいえ，こうなります。

```php
yield 0 => new Member(id: 1);
yield 1 => new Member(id: 2);
yield 0 => new Member(id: 3);
yield 1 => new Member(id: 4);
```

もし PHP がリストと連想配列を区別する言語であれば，以下のようにジェネレータの型も（言語レベルで）区別して 2 つ用意されたかもしれません。

| 正確評価データ型 | 遅延評価データ型                  |
|:---------|:--------------------------|
| リスト      | `Generator<TValue>`       |
| 連想配列     | `Generator<TKey, TValue>` |

ところが，実際は `Generator<TKey, TValue>` （ジェネリクスは言語レベルでは存在しない）の 1 つのみです。 **リストとして使っているつもりの配列を `yield from` に食わせたらキーつきで列挙されてしまいます。**

一方で，修正後のようにキーを省略して `yield` を書くと，複数回呼ばれても連番として一貫性のあるキーを割り振ってくれるようになります。

```php
foreach ($response->members() as $member) {
    yield $member;
}
```

```php
yield 0 => new Member(id: 1);
yield 1 => new Member(id: 2);
yield 2 => new Member(id: 3);
yield 3 => new Member(id: 4);
```

### 修正前のコードで引き起こされる問題

もし修正前のコードを書いたとしても，途中で一回も配列変換せずに `Generator` として最後まで使う場合は問題ありません。

```php
foreach ($this->members() as $i => $member) {
    echo "{$i}: Member({$member->id})\n";
}
```

```
0: Member(1)
1: Member(2)
0: Member(3)
1: Member(4)
```

一度でも配列に変換しようものなら，結果は後勝ちで上書きされてしまいます。

```php
$items = iterator_to_array($this->members());
```
```php
$items = [...$this->members()]; // シンタックスシュガー
```

```
0: Member(3)
1: Member(4)
```

**1 つのジェネレータ関数の中で `yield from` が繰り返し呼ばれる場合，キーの重複に気を配りましょう。**

:::message
別解として，使う側が気をつけるという方向性に倒すのであれば以下のような解決もできます。

```php
$items = iterator_to_array($this->members(), preserve_keys: false);
```

```
0: Member(1)
1: Member(2)
2: Member(3)
3: Member(4)
```

但し一歩間違えばバグを生んでしまうコードになっていることには間違いないので，可能であれば実装を提供する側が配慮するようにしましょう。
:::

[^1]: [PHP: ジェネレータの構文 - Manual # yield from によるジェネレータの委譲](https://www.php.net/manual/ja/language.generators.syntax.php#control-structures.yield.from)
