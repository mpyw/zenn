---
title: "モジュール境界を意識した「なんちゃってクリーンアーキテクチャ」からの進化系"
emoji: "🥳"
type: "tech"
topics: ["laravel", "php"]
published: false
---

# はじめに

以前，以下のような記事で *「なんちゃってクリーンアーキテクチャ」* という，クリーンアーキテクチャの要素を8割ぐらい端折った実装を主張しました。

https://zenn.dev/mpyw/articles/ce7d09eb6d8117

1年少しを経て，反省点などを踏まえ，新たによりよい形のアーキテクチャを提唱しようと思います。

# 以前のアーキテクチャの概要

## コアとなる思想

- **`app/UseCases`** に **単一責務** な UseCase クラスとしてロジックを切り出し， HTTP 層のコントローラや CLI 層に直接ロジックを書くことを避ける。
- データベースアクセスを「インフラ」ではなく「ドメイン」の一部として例外視し，抽象化の責務を免除する。即ち， **UseCase から直接 Eloquent Model 等を扱うことを認める**。

![構成図](https://storage.googleapis.com/zenn-user-upload/nu9y70g1x8ps7ncxw22ib54p7d7m)
*構成図*

## サービスの種別

| 種別 | 置き場所 | 説明 |
|:---|:---|:---|
| **ドメイン**サービス | `app/UseCases` が吸収してくれると考える |ビジネスで実現したいことに関して，中核的なロジックを担うサービス。 |
| **インフラ**サービス | `app/Services` | ネットワークやファイルストレージへのアクセスを担うサービス。 |
| アプリケーションサービス | `app` 配下の Laravel 標準ディレクトリ全域 | HTTP アクセスを捌くサービス。要するにルーティング+コントローラ+コントローラに近い雑多なロジックもろもろ。 |

# 以前のアーキテクチャを運用して感じたこと

## 長所

- ディレクトリ構成が非常にシンプルで，標準的な Laravel プロジェクト経験者から見た学習コストが低い。
- Laravel の Eloquent Model が持つ強力な機能，具体的にはリレーションなどの機能の恩恵を受けることができる。

## 短所

- **ドメイン分離に関して関心が薄く，雑多な気持ちでトランザクションスクリプト的な UseCase が書かれてしまいやすい。**
  - これは「そういうものである」と妥協していましたが， UseCase 層でのクラスの切り分け方が書き手の裁量に依存する状態になっており，レビューするときに「違う，そうじゃないんだ…」と指摘したくなってしまうことが過去にありました。指針不足を感じました。
  - Eloquent Model とドメインモデルが1対1にならない場合は Eloquent Model をラップするクラスを用意してよいとしていますが，これも「指針が分かりづらいので取り敢えず UseCase に全部書いておこう」となりがちな傾向を感じました。
- **UseCase が複雑化したとき，パス分岐のカバレッジを軽量なユニットテストでカバーすることができず，すべてデータベースアクセスを伴う機能テストでカバーする必要が出てきてしまう。**
  - すべての UseCase が複雑になるわけではないし，UseCase にとって重要度が高いのは機能テストであるため必須とまでは言いませんが，補助的に副作用を排除したユニットテストで対応できる場合があると助かるケースはあると感じました。

## 改善したいこと・維持したいこと

- 改善
  - ドメイン分離を意識し，1つのモノリスの中での擬似的なモジュール分割を行いたい。
  - UseCase をユニットテスト可能な状態にしたい。
- 維持
  - Repository を導入することによって， Eloquent Model が持っている強力な機能を封印したくない。
  - Repository を導入することによって，値の詰め替えで必要以上に消耗したくない。

## 触れないこと

- 「境界づけられたコンテキスト」の制約を適用するモジュラモノリス化
  - `composer.json` を複数管理するような本物のパッケージ分割は行わず，あくまで1つのモノリスの中で仮想的にモジュール分割を意識したディレクトリ構成にする。

モジュラモノリスに関しては，弊社のプロジェクトで実際に適用して開発を進めておりますが， **PhpStorm のモジュールサポートが非常に貧弱である**ことに起因して，本来は必要無かった余計な作業工程が大量に発生してきます。ここまでやってもメリットはありますが，一般論として提唱するにはあまりにも難易度が高すぎるため，別の記事でまた紹介しようと考えています。

# アーキテクチャの提唱

<!-- https://tree.nathanfriend.io/?s=(%27options!(%27fancy!true~fullPath!false~trailingSlash!true~rootDot!true)~source!(%27source!%27app9Console9*Commands0KernelYDomain0J0BJV*J2JVJ2J6W-30B3V*32*7V*72*36V3V327V7236V36W-50*ZszNLimitExceededZ-BNV*N2*8V*82*56VNVN28V8256-O0BQV*O6YInfra0O0*QszSlackQV*TwitterQV*FacebookQVGlobalConfigVO6VO6WV9Zs0HandlerYHttp0js0*JzX_43z%22X_45zN%22NXN_4*8%228X8_j-As0*JAV3AV5AVA-Kernel-Middleware%2F9Models0J-3-37-5N-58YWsz%27)~version!%271%27)*%20%20-Y*0%2F9*2RepositoryV3Community4jV5Conversation6Service7MemberProfile8Comment9%5Cn*AControllerB*ContractzJUserNPostOSharingQChannelV-*WProviderXStore4*Y.php9ZException_UpdatejRequestz0**%22Index4*%01%22zj_ZYXWVQONJBA987654320-* -->

以前のサンプルをベースに，少し機能を発展させたバージョンを考えてみます。

```
.
└── app/
    ├── Console/
    │   ├── Commands/
    │   └── Kernel.php
    ├── Domain/
    │   ├── User/
    │   │   ├── Contract/
    │   │   │   ├── User.php
    │   │   │   └── UserRepository.php
    │   │   ├── User.php
    │   │   ├── UserRepository.php
    │   │   └── UserServiceProvider.php
    │   ├── Community/
    │   │   ├── Contract/
    │   │   │   ├── Community.php
    │   │   │   ├── CommunityRepository.php
    │   │   │   ├── MemberProfile.php
    │   │   │   ├── MemberProfileRepository.php
    │   │   │   └── CommunityService.php
    │   │   ├── Community.php
    │   │   ├── CommunityRepository.php
    │   │   ├── MemberProfile.php
    │   │   ├── MemberProfileRepository.php
    │   │   ├── CommunityService.php
    │   │   └── CommunityServiceProvider.php
    │   ├── Conversation/
    │   │   ├── Exceptions/
    │   │   │   └── PostLimitExceededException.php
    │   │   ├── Contract/
    │   │   │   ├── Post.php
    │   │   │   ├── PostRepository.php
    │   │   │   ├── Comment.php
    │   │   │   ├── CommentRepository.php
    │   │   │   └── ConversationService.php
    │   │   ├── Post.php
    │   │   ├── PostRepository.php
    │   │   ├── Comment.php
    │   │   ├── CommentRepository.php
    │   │   └── ConversationService.php
    │   └── Sharing/
    │       └── Contract/
    │           ├── Channel.php
    │           └── SharingService.php
    ├── Infra/
    │   └── Sharing/
    │       ├── Channels/
    │       │   ├── SlackChannel.php
    │       │   ├── TwitterChannel.php
    │       │   └── FacebookChannel.php
    │       ├── GlobalConfig.php
    │       ├── SharingService.php
    │       └── SharingServiceProvider.php
    ├── Exceptions/
    │   └── Handler.php
    ├── Http/
    │   ├── Requests/
    │   │   ├── User/
    │   │   │   ├── StoreRequest.php
    │   │   │   └── UpdateRequest.php
    │   │   ├── Community/
    │   │   │   ├── IndexRequest.php
    │   │   │   ├── StoreRequest.php
    │   │   │   └── UpdateRequest.php
    │   │   └── Conversation/
    │   │       ├── PostIndexRequest.php
    │   │       ├── PostStoreRequest.php
    │   │       ├── PostUpdateRequest.php
    │   │       ├── CommentIndexRequest.php
    │   │       ├── CommentStoreRequest.php
    │   │       └── CommentUpdateRequest.php
    │   ├── Controllers/
    │   │   ├── UserController.php
    │   │   ├── CommunityController.php
    │   │   ├── ConversationController.php
    │   │   └── Controller.php
    │   ├── Kernel.php
    │   └── Middleware/
    ├── Models/
    │   ├── User.php
    │   ├── Community.php
    │   ├── CommunityMemberProfile.php
    │   ├── ConversationPost.php
    │   └── ConversationComment.php
    └── Providers/
```
