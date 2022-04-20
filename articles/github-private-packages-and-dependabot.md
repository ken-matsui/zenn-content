---
title: "会社の GitHub Private Packages を Dependabot に更新させる"
emoji: "🤖"
type: "tech"
topics: ["github", "githubpackages", "npm", "dependabot"]
published: true
---

GitHub Packages を使用して、Private な社内用パッケージを用意している場合、デフォルトの Dependabot 設定では上手くバージョンが更新されません。

そこで、社内用の Private パッケージと Dependabot の組み合わせ方について紹介させていただきます。

# GitHub Packages 用の Personal Access Token を作成する

以下の手順で Personal Access Token を作成し、コピーします。

1. https://github.com/settings/tokens/new へアクセスする
2. `read:packages` のみをチェックして、`Generate token` をクリックする
3. 生成されたトークンをコピーする

会社の Bot 的なユーザーを用意している場合は、そちらから、`read:packages` のトークンを作成しておくと管理しやすいです。

# Dependabot secrets を設定する

Organization のトップページへ移動し、Settings > Security > Secrets > Dependabot へ移動し、`New organization secret` をクリックする。

Name を一旦 `GPR_READ_PACKAGES_TOKEN` とし、先程コピーしておいた、Personal Access Token を貼り付け、`Add secret` をクリックします。

:::message
今回の場合は、Organization secrets として作成しました。
レポジトリ単体で問題無い場合は、Repository secrets として作成してください。
:::

# `.npmrc` を作成する

プロジェクトルートに、以下のように `.npmrc` を作成してください。

```: .npmrc
//npm.pkg.github.com/:_authToken=${GPR_READ_PACKAGES_TOKEN}
@example-inc:registry="https://npm.pkg.github.com"
```

# Dependabot の設定を改善する

先程作成した、Secrets を読み込む形で、設定を変更します。

```yaml diff: .github/dependabot.yml
version: 2

+registries:
+  npm-github:
+    type: npm-registry
+    url: https://npm.pkg.github.com
+    token: ${{ secrets.GPR_READ_PACKAGES_TOKEN }}

updates:
  - package-ecosystem: "npm"
    directory: "/"
+    registries:
+      - npm-github
    schedule:
      interval: "daily"
```

# 最後に

後は問題無く、Dependabot が Private な社内用パッケージのバージョンを更新できるようになっているかと思います。

Renovate の場合は、Personal Access Token を秘匿したままで設定する方法がよくわかりませんでした。
こうやって、GitHub 謹製のサービスを組み合わせていくと、それぞれが相性が良く、セキュアに保てて良いなと感じました。

こちらの記事が参考になれば幸いです。
