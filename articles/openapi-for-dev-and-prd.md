---
title: "開発 & 本番環境用の OpenAPI が肥大化する問題"
emoji: "📋"
type: "idea"
topics: ["openapi", "cloudendpoints"]
published: true
---

Cloud Endpoints 等を運用している場合、認証関係やバックエンドのドメインが、開発環境と本番環境で違うため、どうしても開発環境と本番環境それぞれで、`openapi.yaml` を用意する必要があります。

https://zenn.dev/matken/articles/cloud-endpoints

基本的な運用方法としては、`dev/openapi.yaml` と `prd/openapi.yaml` を用意することだと思います。
すると、少しの変更でも両方とも編集する必要があり、API の変更や追加がどんどん重荷になっていきます。

Cloud Endpoints を使用していると、その編集したドキュメントがそのままプロキシの挙動に繋がるため、少しでもミスをしていると、バックエンド側ではキチンと実装しているのに上手く動作しないみたいなことが起きます。
さらにプロダクトが肥大化してくると、`openapi.yaml` 自体が大きすぎて、Jetbrains 系の IDE だとそのファイルを開いたら解析が走ってしまい、しばらく反応が無くなる始末でした。
代わりに VS Code を使用してはいましたが、YAML で書くため、インデントも厳しく、編集する度に非常に細かく何度も両方とものファイルが間違っていないかチェックしていました。

そういったこともあり、実際に運用していた時、API の変更は結構キツかったです。

そこで、そういった負担を減らすため、以下 2 点の改善をしました。

1. 開発環境用と本番環境用を統一化する
2. `swagger-cli bundle` を使用して結合する

# 1. 開発環境用と本番環境用を統一化する

私の場合は、開発環境の `openapi.yaml` を起点に本番環境用の `openapi.yaml` を生成するという方法を取りました。

認証関連の情報や、バックエンドに関する情報での、開発環境用と本番環境用との差分は、ドメインのみでした。
それに加え、サブドメインの命名規則も統一していた（認証であれば、`auth.example.(dev|app)` のような感じ）ため、ドメインの差し替えだけで済むことが判明しました。

そこで、`dev/openapi.yaml` のみを配置し、npm script として `sed` コマンドを実行することで、本番環境用の `openapi.yaml` を生成できました。

```json: package.json
{
  "scripts": {
    "build:prd": "sed 's/example.dev/example.app/g' dev/openapi.yaml > prd/openapi.yaml"
  }
}
```

他にも、dev という文言があって、それを prd と置き換えたい場合は以下のように、セミコロンで繋げることができます。

```json: package.json
{
  "scripts": {
    "build:prd": "sed 's/example.dev/example.app/g; s/dev/prd/g' dev/openapi.yaml > prd/openapi.yaml"
  }
}
```

# 2. `swagger-cli bundle` を使用して結合する

開発環境用と本番環境用を分割していても、どうしても肥大化してしまいます。
IDE が重すぎて動かないレベルになってくると最悪です。

そこで、path 単位等でファイルを分割し、`$ref` 記法を使用してそれを参照しておくことで、`swagger-cli bundle` コマンドで 1 つの巨大な `openapi.yaml` を生成できます。

`swagger-cli bundle` の詳しい使い方に関しては、以下の記事が参考になります。

https://garafu.blogspot.com/2020/06/multi-file-openapi.html

分割した場合は、例えば `dev/index.yaml` と置いておき、`swagger-cli` を使用して `dev/openapi.yaml` に結合後の内容を出力します。
それで開発環境用は完成で、それに対して、`sed` コマンドを実行する先程のコマンドを実行することで、本番環境用も完成します。

```json: package.json
{
  "scripts": {
    "build:dev": "npx @apidevtools/swagger-cli bundle -o dev/openapi.yaml -t yaml dev/index.yaml",
    "build:prd": "npm run build:dev && sed 's/example.dev/example.app/g; s/dev/prd/g' dev/openapi.yaml > prd/openapi.yaml"
  }
}
```

後は、それぞれの `dev/openapi.yaml` と `prd/openapi.yaml` を Cloud Endpoints へデプロイする等の作業が行えます。

# 最後に

こうすることで肥大化に強い OpenAPI ドキュメントが完成しました。
どうしても肥大化してしまうので、早い段階で行うことをおすすめします。
かなり大きくなった後に分割するのは、かなりキツイです。

この記事が参考になれば、幸いです。
