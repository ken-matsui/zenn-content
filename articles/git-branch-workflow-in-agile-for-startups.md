---
title: "初期スタートアップでのアジャイル開発における Git ワークフロー"
emoji: "🔁"
type: "idea"
topics: ["git", "github", "agile"]
published: true
---

初期のスタートアップで、初めてのプロダクトではウォーターフォールで開発し、その後アジャイル開発に転換していくことが多いんじゃないかと思います。
その開発スタイルに合わせて、Git ワークフローを適宜変更していけると開発しやすい環境づくりに繋がっていきます。

ウォーターフォールにおける Git ワークフローに関しては、以下の記事で紹介させていただきました。

https://zenn.dev/matken/articles/git-workflow-in-waterfall-for-startups

こちらの記事では、アジャイル開発における Git ワークフローに関して 1 つの案として紹介させていただきます。

# Git ブランチワークフローを定義する

ウォーターフォールでの Git ワークフローは、GitHub flow をベースにしているため、非常にシンプルでした。

![ウォーターフォール開発のための Git ブランチワークフロー](/images/git-workflow-in-waterfall-for-starups/git-branch-workflow-for-waterfall.jpg)
*Source: [スタートアップでのウォーターフォール開発における Git ワークフロー](https://zenn.dev/matken/articles/git-workflow-in-waterfall-for-startups)*

しかし上記のウォーターフォール用のフローだと、何かの機能の開発途中で一旦 `main` にマージしていたりして、そこから `hotfix` が生えた時、すぐさま `hotfix` のデプロイに進むことができません。

アジャイル開発においては、安定した開発と、部分的なデプロイに対応する必要があります。
そこで、以下のブランチワークフローを作成してみました。

![Git ブランチワークフロー](/images/git-branch-workflow-in-agile-for-startups/git-workflow.png)

## 特徴

特徴は以下の通りです。

* [git-flow](https://nvie.com/posts/a-successful-git-branching-model/) ベース
* `hotfix` の迅速で部分的な取り込み & デプロイが可能
* Jira 等のアジャイル開発における、チケットタイプと統合
  * `hotfix/*`: Bug
  * `develop/*`: Story & Task
  * `feature/*`: Sub-Task
* Story や Task 単位での部分的な取り込み & デプロイに対応
* チケットタイプと統合したため、フロー自体を理解しやすい
* `staging` を今は用意しなくても、後から追加しやすい

またウォーターフォールでの Git ワークフローは、実質 queue なので別の実装に依存しているだとか、前のスプリントの開発が絶妙に残っているとかが原因で、デプロイが詰まる時があります。

そこを、スプリントが複数跨ぐ場合でも、選択的なデプロイによって、安定してプロダクトを新鮮に保つことができます。

## ブランチとクラウド環境の対応

それぞれのブランチをクラウドの環境と対応付けておくと、より扱いやすくなります。
例えば、以下のような運用が考えられます。

* `main` ブランチでのタグ: Production 環境
* `staging` ブランチでのコミット: Staging 環境
* それ以外のブランチでのコミット: Preview 環境 (Vercel や Netlify の Preview のようなもの)

# 最後に

共同開発において、どのように Git ブランチワークフローを定義するのかは悩みどころだと思います。

そういった中で、こちらの記事が 1 つのアイデアになれば、幸いです。
