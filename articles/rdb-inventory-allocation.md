---
title: "数量限定商品の在庫引当処理のスケーラビリティを考慮した実装の検討"
emoji: "🛒"
type: "tech"
topics: ["postgresql", "postgres", "mysql", "database", "rdb"]
published: false
---

# この記事のテーマ

オンラインショッピングサイトにおける，数量限定商品（早いもの勝ち商品）の在庫引当処理を検討する。

![](https://storage.googleapis.com/zenn-user-upload/c810fd18b6ed-20220702.png)
![](https://storage.googleapis.com/zenn-user-upload/4724669aba7c-20220702.png)

画面に残り在庫数が表示されていて，在庫が確保できる範囲で先着順に注文を確定させていく。残り在庫が無くなった時点で注文を打ち切りとする。

:::message
- ベンチマークは取っておりますが，厳密な結果の掲載は省略します。
- 全てにおいてこの記事の内容が当てはまるわけではありません。ご自身で計測された上で最終判断されることをおすすめします。
:::

## 読者対象

- ある程度データベースに関する知識を持っている，経験年数 1 年以上のバックエンドエンジニア
- 特定のプログラミング言語に依存する部分は最小に抑え，すべての SQL 使用者を対象とする

## RDBMS の対象バージョン

- PostgreSQL: 9.5 以降
- MySQL 8.0 以降

# 比較検討

まず，参考文献として以下の記事を当たった。2 つの記事とも，初期状態では **在庫数をデータベースに直接持って** 状態管理している。

- [大量アクセスに耐え得る在庫管理システムの構成を考え実装してみた | Basicinc Enjoy Hacking!](https://tech.basicinc.jp/articles/155)
  - 機能に乏しい古いバージョンの MySQL 5.x 系を使っていると思われるが， **1 在庫ごとに 1 レコードを作る** 案に到達している。
- [BOOTHの“決済スパイク”を防げ！　創作物の総合マーケットを支えるトランザクション分割 - ログミーTech](https://logmi.jp/tech/articles/324990)
  - 1 在庫ごとに 1 レコードを作る案は検討されているが，取り扱う商品の特性上，在庫数が多すぎて見送られている。結果， **在庫管理トランザクションと決済トランザクションを分割する** ことで対処されている。

2 つの記事はかなり異なる結論に到達しており，どれが最も優れていると判断できるかは要件次第で大きく変わると言える。ここでは，以下の観点別に考え方を比較していく。 BOOTH の記事で検討されていた非同期の決済については，ユーザ体験があまり一般的ではないと思われるのでここでは触れない。

## 在庫数の管理方法

以下の 2 つを考える。

### A. ステートフル方式: 残り在庫数をテーブルに保持

データ構造的には誰もが考えつく，オーソドックスな方式。以下の例では，商品 P1 の残り在庫数が 3 となっている。

| 商品ID | 商品名   | 総在庫数 | 残り在庫数 |
|:----:|:------|:----:|:-----:|
|  P1  | 高級時計  |  5   | **3** |
|  P2  | 超高級時計 |  1   |   1   |

| 注文ID | ユーザID | 商品ID | 注文数 |
|:----:|:-----:|:----:|:---:|
|  O1  |  U1   |  P1  |  2  |

極めてシンプルに見えるが，決定的な欠点を持つ。**残り在庫数の信頼性を保つためにはレコードの悲観ロックがほぼ必須で，悲観ロックした 1 レコードにアクセスが集中する**というのが最大の欠点となる。

以下で， RDBMS 別の SQL の実装イメージを示す。

:::message alert
以下の例ではまだ，ショッピングカートでの保留や決済処理は考慮していません。ショッピングカートの概念は存在せず，決済処理なしに即座に注文が完了するものと仮定しています。
:::

:::message
在庫数が希望注文数に満たない場合，可能なぶんだけを注文する仕様とします。例えば残り在庫数が 3 で希望注文数が 5 である場合， 3 個分は注文するとします。
:::

#### Postgres の場合

Postgres では `NO KEY` を選択して `FOR UPDATE` のロックレベルを少し下げることが出来るが，今回はそれほど役には立たない。単一行に対する集中的な悲観ロックは大きなオーバーヘッドになるだろう。

```sql
BEGIN;
    
WITH
対象商品 AS (
    SELECT * FROM 商品
    WHERE 商品ID = :商品ID AND 残り在庫数 > 0
    FOR NO KEY UPDATE
),
可能注文数 AS (
    SELECT LEAST((SELECT 残り在庫数 FROM 対象商品), :希望注文数)
),
生成する注文レコード AS (
    UPDATE 商品 SET 残り在庫数 = 残り在庫数 - (SELECT * FROM 可能注文数)
    WHERE 商品ID = :商品ID
    RETURNING :ユーザID, :商品ID, (SELECT * FROM 可能注文数)
)
INSERT INTO 注文(ユーザID, 商品ID, 注文数)
SELECT * FROM 生成する注文レコード

COMMIT;
```

簡易的に，「残り在庫数」更新後に「注文」を連続して生成しているが，この間にアプリケーション側で別の処理が入ってくることも考えられる。挟まれる処理をいかに短くするかが課題となる。

#### MySQL の場合

MySQL では `FOR UPDATE` のレベルを下げられないことに加えて `RETURNING` も使えない制約があるので， SQL だけで完結させる場合は以下のように変数への格納を併用する必要がある。

```sql
BEGIN;

WITH 対象商品 AS (
    SELECT * FROM 商品
    WHERE 商品ID = :商品ID AND 残り在庫数 > 0
    FOR UPDATE
)
UPDATE 商品
SET 残り在庫数 = 残り在庫数 - @可能注文数
WHERE (@可能注文数 := LEAST(
    :希望注文数,
    COALESCE((SELECT 残り在庫数 FROM 対象商品), 0)
)) > 0 AND 商品ID = :商品ID;

INSERT INTO 注文(ユーザID, 商品ID, 注文数)
SELECT * FROM :ユーザID, :商品ID, @可能注文数
WHERE @可能注文数 > 0

COMMIT;
```

本質的な部分は Postgres の場合と同じで，抱えている問題も同等である。

:::message alert
MySQL はトランザクション分離レベルとして **`READ UNCOMMITTED`** を選択できるため，失敗時のロールバック保証のためデータベーストランザクションを貼りつつ，まだコミットされていない最新の残り在庫数を参照させることができます。

- [READ UNCOMMITTEDが有効なシーン - ほげにっき](https://hogedigo.hatenablog.jp/entry/20090429/1240985095)

しかし以下の情報によると， WHERE で判断した条件は悲観ロックなしでは UPDATE するタイミングで保証されていないようです。

- [(mysql innodb) Is single update statement with "where" transaction safe?](https://dba.stackexchange.com/questions/175228/mysql-innodb-is-single-update-statement-with-where-transaction-safe/175247#175247)

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN;

UPDATE 商品 SET 残り在庫数 = 残り在庫数 - :希望注文数
WHERE 商品ID = :商品ID AND :希望注文数 > 残り在庫数;

INSERT INTO 注文(ユーザID, 商品ID, 注文数)
SELECT :ユーザID, :商品ID, :希望注文数 FROM DUAL
WHERE ROW_COUNT() > 0;

COMMIT;
```
:::


### B. ステートレス方式: 1在庫=1レコードとして在庫テーブルを作る

以下の例では，商品 P1 の残り在庫数が 3 となっている。確保したユーザが NULL であれば在庫として残っていることを意味する。

| 商品ID | 商品名   |
|:----:|:------|
|  P1  | 高級時計  |
|  P2  | 超高級時計 |

| 在庫ID | 商品ID | 確保したユーザID |
|:----:|:----:|:---------:|
|  S1  |  P1  |    X1     |
|  S2  |  P1  |    X2     |
|  S3  |  P1  | **NULL**  |
|  S4  |  P1  | **NULL**  |
|  S5  |  P1  | **NULL**  |
|  S6  |  P2  |   NULL    |

以下の注文テーブルは，オプションとなる。数量限定商品だけのために設計する場合は不要だが，そうではない商品も兼ねる場合は必要となるだろう。

| 注文ID | ユーザID | 商品ID | 注文数  |
|:----:|:-----:|:----:|:----:|
|  O1  |  U1   |  P1  |  2   |

#### Postgres の場合

```sql
BEGIN;

WITH 生成する
SELECT * FROM 商品在庫(
    WHERE 商品ID = :商品ID
    LIMIT :希望注文数
    FOR NO KEY UPDATE SKIP LOCKED NO WAIT
)
    
WITH 対象商品 AS (
    SELECT * FROM 商品
    WHERE 商品ID = :商品ID AND :希望注文数 > 残り在庫数
    FOR NO KEY UPDATE
),
WITH 生成する注文レコード AS (
    UPDATE 商品 SET 残り在庫数 = 残り在庫数 - :希望注文数
    WHERE id = (SELECT id FROM 対象商品)
    RETURNING :ユーザID, :商品ID, :希望注文数
)
INSERT INTO 注文(ユーザID, 商品ID, 注文数)
SELECT * FROM 生成する注文レコード

COMMIT;
```

#### MySQL の場合

## 在庫消費のタイミング

### C. 在庫確保から決済までを 1 つのトランザクションで一気に行う

### D. 在庫確保トランザクションと決済トランザクションに分割する

## 観点別の評価

### 決済処理を




