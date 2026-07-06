---
title: "Git 感覚で AWS / Google Cloud / Azure のシークレットを管理できる CLI/GUI ツール「suve」を作った"
emoji: "🔐"
type: "tech"
topics: ["aws", "googlecloud", "azure", "secretsmanager", "cli"]
published: true
publication_name: "yumemi_inc"
---

# はじめに

Agentic AI 全盛期の昨今，コードを書くのも，テストを書くのも，リファクタリングするのも，かなりの部分を AI に任せられるようになりました。ところが，そんな時代においても **シークレットや設定値の管理だけは，いまだに手作業でクラウドコンソールをぽちぽちするのが普通** ではないでしょうか？

- AWS マネジメントコンソールにログインして，SSM Parameter Store の画面を開いて，該当のパラメータを探して，「編集」ボタンを押して，テキストエリアに値を貼り付けて，保存する
- 「あれ，この値いつ誰が変えたんだっけ？前の値なんだったっけ？」となっても，履歴画面は貧弱で差分も見えない
- 本番環境のシークレットを直接編集するので，**保存ボタンを押す瞬間が毎回怖い**

コード管理されない部分のインフラ構成設定，そんなに高頻度ってわけでもないんですが，いざ設定するときは毎回不安になります。コードの変更なら `git diff` で差分を確認して，レビューを通して，安心してマージできるのに，**シークレットの変更だけはノーガードで本番に直接反映される**。この非対称性をずっと何とかしたいと思っていました。

`aws-cli` が使える環境なら，そのまま Git みたいにパパっと作業したくないですか？ `git log` で履歴を見て， `git diff` で差分を確認して，ステージングしてから反映したくないですか？あわよくば使いやすい GUI も欲しくないですか？

というわけで，作りました。

https://github.com/mpyw/suve

![CLI Demo](https://media.githubusercontent.com/media/mpyw/suve/main/demo/cli-demo.gif)
![GUI Demo](https://media.githubusercontent.com/media/mpyw/suve/main/demo/gui-demo.gif)

# suve とは

**suve**（**S**ecret **U**nified **V**ersioning **E**xplorer）は，クラウド上のシークレット・設定値を Git ライクなインターフェースで管理できる CLI/GUI ツールです。以下のバックエンドに対応しています。

| バックエンド | コマンド | バージョン管理 |
|:---|:---|:---|
| AWS SSM Parameter Store | `suve aws param` | ✅ 数値バージョン |
| AWS Secrets Manager | `suve aws secret` | ✅ UUID + ステージングラベル |
| Google Cloud Secret Manager | `suve gcloud secret` | ✅ 整数バージョン |
| Azure Key Vault | `suve azure secret` | ✅ 不透明バージョン ID |
| Azure App Configuration | `suve azure param` | ❌ バージョンなし |

特徴を一言でまとめると，以下のようになります。

- **Git ライクなコマンド体系**： `show` `log` `diff` `ls` `tag` `stash` など，Git ユーザーなら説明不要のコマンド群
- **ステージングワークフロー**： `edit` → `status` → `diff` → `apply` で，変更内容をローカルで査読してからクラウドに反映
- **バージョンナビゲーション**： `#VERSION` `~SHIFT` `:LABEL` 構文で過去バージョンを自在に参照
- **マルチクラウド対応**：AWS / Google Cloud / Azure を統一された UX で操作
- **GUI モード**： `--gui` フラグでデスクトップアプリとしても起動可能

# インストール

Homebrew（macOS / Linux）の場合：

```bash
# フル版（CLI + GUI）
brew install mpyw/tap/suve

# CLI のみ（GUI 依存なし，Linux ではこちらを推奨）
brew install mpyw/tap/suve-cli
```

Scoop（Windows）の場合：

```powershell
scoop bucket add mpyw https://github.com/mpyw/scoop-bucket.git
scoop install suve
```

Go ユーザーなら `go install` でも入ります（CLI のみ）：

```bash
go install github.com/mpyw/suve/cmd/suve@latest
```

認証は各クラウドの標準的な認証チェーンにそのまま乗っかります。AWS なら環境変数・共有認証ファイル・IAM ロール，Google Cloud なら Application Default Credentials，Azure なら DefaultAzureCredential です。**つまり `aws-cli` や `gcloud` や `az` が既に動いている環境なら，追加の設定なしでそのまま動きます。**

:::message
[aws-vault](https://github.com/99designs/aws-vault) ユーザーなら `aws-vault exec my-profile -- suve param show /my/param` のようにラップするだけで一時クレデンシャルと組み合わせられます。
:::

# 基本的には GUI がおすすめ

以下では CLI 文脈で説明しますが，実際使ってみると GUI のほうが遥かに快適なので，基本的には `--gui` オプションを付与することをおすすめします。 [Wails](https://wails.io/) 製のデスクトップアプリが立ち上がり，一覧・履歴・diff の確認から編集・ステージングまで，CLI と同じ操作を GUI 上でそのまま行えます。

```bash
suve --gui
```

:::message
認証情報は CLI の環境変数（`AWS_PROFILE` など）から拾うことを想定しているので，スタンドアロンな GUI アプリとしてはビルドしていません。**「CLI 経由で起動可能な GUI アプリケーション」** だと思ってください。
:::

# 基本操作：show / log / diff

ここからは AWS（SSM Parameter Store と Secrets Manager）を軸に紹介していきます。 `suve param` が Parameter Store， `suve secret` が Secrets Manager に対応します。他のクラウドでもコマンド体系は同じです（後述）。

## show：値をメタデータ付きで表示

```shellsession
$ suve param show /app/config/database-url
Name: /app/config/database-url
Version: 3
Type: SecureString
Modified: 2024-01-15T10:30:45Z

  postgres://db.example.com:5432/myapp
```

`--raw` を付ければ生の値だけが出力されます。

```bash
# スクリプト内で値を取得
DB_URL=$(suve param show --raw /app/config/database-url)

# ファイルに書き出し
suve param show --raw /app/config/ssl-cert > cert.pem
```

## log：バージョン履歴を表示

`git log` と同じ感覚で，パラメータの変更履歴を一覧できます。

```shellsession
$ suve param log /app/config/database-url
Version 3 (current)
Date: 2024-01-15T10:30:45Z
postgres://db.example.com:5432/myapp...

Version 2
Date: 2024-01-14T09:20:30Z
postgres://old-db.example.com:5432/myapp...

Version 1
Date: 2024-01-13T08:10:00Z
postgres://localhost:5432/myapp...
```

そして真骨頂は `--patch`（`-p`）オプションです。 `git log -p` と同様に，**各バージョンで何が変わったのかを Unified Diff 形式で表示** してくれます。

これが特に威力を発揮するのが Secrets Manager です。Secrets Manager には，DB クレデンシャルや API キーなど **複数の値を 1 つの JSON にまとめて格納する** ことが多いですよね。この運用でありがちなのが，「バージョン間で一部のフィールドだけが変わる」ケースです。JSON は 1 行に詰め込まれて格納されていることも多く，素の diff では **行全体が変わったようにしか見えず，どのフィールドが変わったのか分かりません**。

そこで `--parse-json`（`-j`）の出番です。JSON を整形し，キーをアルファベット順にソートしてから diff を取ってくれるので，フォーマットの揺れに惑わされず **変わったフィールドだけが浮かび上がります**。

これらのオプションを組み合わせると…？

```shellsession
$ suve secret log --patch --parse-json myapp/credentials
```

```diff
Version e7c1b2a3 (AWSCURRENT)
Date: 2024-01-15T10:30:45Z

--- myapp/credentials#9f8e7d6c
+++ myapp/credentials#e7c1b2a3
@@ -1,6 +1,6 @@
 {
   "api_key": "sk_live_xxxxxxxx",
   "db_host": "db.example.com",
-  "db_password": "0ld-p@ssw0rd",
+  "db_password": "n3w-p@ssw0rd",
   "db_username": "myapp"
 }
```

パスワードのローテーションで `db_password` だけが変わったことが一目瞭然です。「この値いつ誰が変えた？前の値は？」問題は，これで完全に解決です。

:::message
`--parse-json` は `log` だけでなく `show` や `diff`，後述するステージングの `stage diff` でも使えます。もちろん Secrets Manager 専用ではなく，全バックエンドで共通のオプションです。
:::

## diff：バージョン間の差分を表示

一番よく使うのは「最新版と 1 つ前の版の比較」でしょう。バージョン指定構文 `~`（チルダ）を使えば一発です。

```shellsession
$ suve param diff /app/config/database-url~
```

```diff
--- /app/config/database-url#2
+++ /app/config/database-url#3
@@ -1 +1 @@
-postgres://old-db.example.com:5432/myapp
+postgres://db.example.com:5432/myapp
```

任意の 2 バージョン間の比較も可能です。

```shellsession
$ suve param diff /app/config/database-url#1 /app/config/database-url#3
```

## バージョン指定構文

Git の `HEAD~2` や `main^` に慣れている方なら，直感的に使えるはずです。

| 構文 | 意味 |
|:---|:---|
| `/my/param` | 最新バージョン |
| `/my/param#3` | バージョン 3 |
| `/my/param~1` | 1 つ前のバージョン |
| `/my/param#5~2` | バージョン 5 の 2 つ前 = バージョン 3 |
| `/my/param~~` | 2 つ前（`~1~1` と同義） |
| `my-secret:AWSPREVIOUS` | ステージングラベル指定（AWS Secrets Manager のみ） |

AWS Secrets Manager は数値バージョンではなく UUID + ステージングラベルという独自体系ですが，suve では `~1` のような相対指定がそのまま使えるので，**バックエンドごとのバージョン体系の違いを意識する必要がほとんどありません**。

# ステージングワークフロー：本番直編集の恐怖からの解放

さて，ここが suve の目玉機能です。クラウドコンソールでの直接編集が怖い理由は，**「変更内容を反映前に確認するステップが存在しない」** ことに尽きます。Git なら `git add` → `git diff --staged` → `git commit` という流れで変更を査読できるのに，シークレット管理にはそれがない。

suve はこのワークフローをそのまま持ち込みました。

## 1. 変更をステージする

```shellsession
$ suve stage param add /app/config/new-param "my-value"
✓ Staged for creation: /app/config/new-param

$ suve stage param edit /app/config/database-url
✓ Staged: /app/config/database-url

$ suve stage param delete /app/config/old-param
✓ Staged for deletion: /app/config/old-param
```

`edit` は `$EDITOR`（または `$VISUAL`）で設定したエディタが開きます。VSCode 派なら `export VISUAL='code --wait'` を設定しておきましょう。

**この時点ではまだクラウドには何も反映されていません。** 変更はローカルのステージングエリアに溜まっているだけです。

## 2. ステージした変更を査読する

```shellsession
$ suve stage status
Staged SSM Parameter Store changes (3):
  A /app/config/new-param
  M /app/config/database-url
  D /app/config/old-param
```

`git status` の `A` / `M` / `D` 表記そのままですね。そして `diff` で実際の差分を確認します。

```shellsession
$ suve stage diff
```

```diff
--- /app/config/database-url#3 (AWS)
+++ /app/config/database-url (staged)
@@ -1 +1 @@
-postgres://db.example.com:5432/myapp
+postgres://new-db.example.com:5432/myapp

--- /app/config/new-param (not in AWS)
+++ /app/config/new-param (staged for creation)
@@ -0,0 +1 @@
+my-value

--- /app/config/old-param#2 (AWS)
+++ /app/config/old-param (staged for deletion)
@@ -1 +0,0 @@
-deprecated-value
```

**反映前に，全変更が Unified Diff で一覧できる。** これがやりたかったんです。

## 3. 変更を反映する

```shellsession
$ suve stage apply
Applying SSM Parameter Store parameters...
✓ Created /app/config/new-param
✓ Updated /app/config/database-url
✓ Deleted /app/config/old-param
```

`apply` は実行前に確認プロンプトを出します。間違えてステージした場合は `suve stage param reset` で個別に，`suve stage reset --all` で全部取り消せます。

## stash：作業の一時退避

「変更を準備したけど，反映は明日のメンテナンスウィンドウで」みたいな場面のために， `git stash` 相当の機能もあります。

```bash
suve stage stash        # ステージ内容をファイルに退避（パスフレーズを設定）
suve stage stash show   # 退避内容をプレビュー
suve stage stash pop    # 復元
suve stage stash drop   # 破棄
```

## セキュリティ面の配慮

「シークレットをローカルに溜め込むのは怖くない？」という懸念はもっともです。suve では以下のように対策しています。

- ステージング状態は **OS のキーチェーンに保存したデータキーで暗号化** して保持（`~/.suve/staging/` 配下）
- stash は **Argon2 + AES-GCM によるパスフレーズベースの暗号化** を別途適用
- キーチェーンが使えない環境では `SUVE_STAGING_KEY` 環境変数によるキー指定も可能

# マルチクラウド対応：「うちの会社，基本 Azure なのよね…」

実は suve は最初 AWS 専用ツールとして作っていました。せっかく作ったので，インフラエンジニアの知り合いに「こんなの作ったよー！」と宣伝したところ，返ってきた言葉が

**「うちの会社，基本 Azure なのよね…」**

でした。悔しかったので Google Cloud Secret Manager，Azure Key Vault，Azure App Configuration まで対応させました。今では大抵のプロジェクトで役に立てるはずです。

## どのクラウドでも同じ UX

明示的なコマンドグループは常に利用可能です。

```bash
suve aws param     ...  # AWS SSM Parameter Store
suve aws secret    ...  # AWS Secrets Manager
suve gcloud secret ...  # Google Cloud Secret Manager
suve azure secret  ...  # Azure Key Vault
suve azure param   ...  # Azure App Configuration
```

`show` / `log` / `diff` / `list` / ステージングといった操作体系は全バックエンド共通なので，**「AWS の案件で覚えた操作が，そのまま Azure の案件でも通用する」** のがポイントです。

また，各サービスには **普段呼び慣れている名前のエイリアス** をちゃんと用意しています。「Parameter Store のことは SSM って呼んでる」「Key Vault は kv でしょ」という人も，いつもの名前でそのまま叩けます。

```bash
suve aws ps ...  # AWS SSM Parameter Store（param / ps / ssm）
suve aws sm ...  # AWS Secrets Manager（secret / sm）
suve gcp sm ...  # Google Cloud (google / gcp) Secret Manager (secret / sm)
suve az kv  ...  # Azure Key Vault (secret / kv)
suve az ac  ...  # Azure App Configuration (param / appconfig / appcfg / ac)
```

## bare alias：環境から文脈を推論

さらに，環境変数から対象クラウドが一意に定まる場合は，プロバイダ名を省略した **bare alias** が使えます。

| 環境 | `param` → | `secret` → | `stage` → |
|:---|:---:|:---:|:---:|
| `AWS_PROFILE` 設定時 | `aws` | `aws` | `aws` |
| `GOOGLE_CLOUD_PROJECT` 設定時 | — | `gcloud` | `gcloud` |
| `AZURE_KEYVAULT_NAME` 設定時 | — | `azure` | `azure` |
| `AWS_PROFILE` + `GOOGLE_CLOUD_PROJECT` | `aws` | —（曖昧） | —（曖昧） |

つまり `AWS_PROFILE` を設定した状態なら，本記事のこれまでの例のように `suve param show ...` と短く書けます。ここで重要な設計判断として，**複数のバックエンドが有効な場合，suve は勝手に優先順位をつけて解決したりせず，alias 自体を無効化します**。シークレットを扱うツールで「意図と違うクラウドに書き込んでしまった」は絶対に起こしてはならないので，**曖昧さを暗黙に解決しない** ことを徹底しています。

# Agentic AI との相性：human-in-the-loop なシークレット運用

冒頭の話に戻りましょう。「AI 全盛期なのにシークレット管理は手作業」という課題に対して，suve は 2 つの角度から答えを持っています。

- 1 つ目は単純で，**手作業のダルさそのものを解消する** こと。
- 2 つ目はもう一歩踏み込んで，**AI エージェントにシークレット管理作業を部分的に任せられるようになる** ことです。

ここで効いてくるのが，ステージングワークフローの存在です。例えばこんな運用が考えられます。

> **人間: 「staging 環境の Parameter Store に，.env に追加した 3 つの設定値を suve でステージしておいて。値は仮でいいよ」**
&nbsp;
> AI: 「了解です。suve stage param add でステージングしますね」
> ```shell
> suve stage param add /staging/app/feature-x-enabled "false"
> suve stage param add /staging/app/feature-x-timeout "30"
> suve stage param add /staging/app/feature-x-endpoint "https://..."
> ```
> → 「3 件ステージしました。確認してください」
&nbsp;
> **（GUI 派）人間: `suve --gui` でステージング画面を開き，差分を確認して問題なければ「Apply」ボタンを押す**
>
> **（CLI 派）人間: `suve stage diff` で差分を確認し，問題なければ自分で `suve stage apply` を実行**

つまり **「準備は AI，最終承認は人間」という human-in-the-loop の境界線を，ステージングエリアがそのまま提供してくれる** わけです。 
（`apply` には確認プロンプトがあり，スキップには明示的な `--yes` フラグが必要なので，エージェント側の権限設定で `apply` や `--yes` をブロックしておけば，「AI が勝手に本番のシークレットを書き換える」事態は構造的に防げます）

# まとめ

- **シークレット管理のためにクラウドコンソールへログインしてぽちぽちする作業は，suve でターミナルから解放できる**
- `show` / `log` / `diff` と Git ライクなコマンドで，**「いつ・何が変わったか」が差分付きで追える**
- **`edit` → `status` → `diff` → `apply` のステージングワークフロー** で，本番直編集の恐怖から解放される
- **AWS / Google Cloud / Azure に対応** しているので，大抵のプロジェクトで同じ UX が使い回せる
- 日常使いには `--gui` で起動する **デスクトップアプリ** が快適（CLI 経由で起動可能な GUI アプリケーション）
- CLI + ステージングという構造は **AI エージェントに作業を任せつつ人間が最終承認する運用** とも相性が良い

`brew install mpyw/tap/suve-cli` で 10 秒で試せるので，日々コンソールぽちぽちに疲れている方はぜひ触ってみてください。フィードバックや Issue も歓迎です！

https://github.com/mpyw/suve
