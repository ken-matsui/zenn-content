---
title: "GitHub Actions による自動デプロイに承認機能を付ける"
emoji: "👍"
type: "tech"
topics: ["github", "cicd", "githubactions"]
published: true
---

GitHub で共同開発をしている場合、GitHub Actions を使用して継続的なデプロイを行っていることが多いかと思います。
例えば Git タグを使用して、そのタグ作成をフックに自動で Production へデプロイといった運用が考えられます。

ただ、そういった運用は、運用自体が簡単ではあっても、セキュリティに難があります。
業務委託等の、社外エンジニアが開発に参加している場合でも、開発に参加するには、そのレポジトリに対して Write 以上の権限が必要になります。

すると、Git タグを作成する権限もあるわけで、そうなると、社外メンバーが Production デプロイを勝手に行えてしまいます。

そういったことを防ぐために、Production デプロイ自体に承認機能を付けることができます。
それを、GitHub Actions と GitHub Environments という機能を組み合わせることで実現できます。

https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment

# Environment を作成する

:::message
Private レポジトリで Environment を使用するには、GitHub Enterprise を契約する必要があります。
:::

1. レポジトリのページから、Settings > Environments へアクセスする
   ![Step 1](/images/approve-deployments-with-github-environments/step1.png)
2. `New environment` をクリックして、`Production` 等を入力する
   ![Step 2](/images/approve-deployments-with-github-environments/step2.png)
3. `Required reviewers` を有効にし、社内メンバーを追加し、`Save protection rules` をクリックする
   ![Step 3](/images/approve-deployments-with-github-environments/step3.png)
4. `Deployment branches` で、`Selected branches` を選択し、`v*` タグを指定する
   ![Step 4-1](/images/approve-deployments-with-github-environments/step4-1.png)
   ![Step 4-2](/images/approve-deployments-with-github-environments/step4-2.png)

:::message
`Deployment branches` には、ブランチだけではなく、タグも指定できます。
しかし、`Currently applies to 0 branches` とあるように、そのタグが存在していても、カウントされないようです。
:::

# GitHub Actions を作成する

例えば、以下の GitHub Actions を作成してみます。

```yaml
name: 'Deploy'

on:
  push:
    tags:
      - 'v*'

jobs:
  deploy:
    environment:
      name: Production

    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3.0.0
      - run: echo 'Hello'
```

`environment` のところで、先程作成した Environment の名前を指定します。

`steps` の内容は、本来は、Cloud Run へのデプロイ等の処理が入ってきます。

また、GitHub Environments は、`steps` の中の処理が失敗しないかしか見ていません。

# デプロイする

上記の GitHub Actions は、tag で実行されます。

そのため、以下のようにローカルでタグを作成して、push してみます。

この時、先程作成した `Deployment branches` のルールに従って、`v` から始める必要があります。

```bash
$ git tag v0.1.0
$ git push origin v0.1.0
```

上記を実行すると、以下のように Actions ページで承認用の画面が表示されます。

![Action](/images/approve-deployments-with-github-environments/action.png)

後は、`Review pending deployments` をクリックすると、以下のように承認画面に進みます。

![Approval](/images/approve-deployments-with-github-environments/approval.png)

ここで、チェックボックスをクリックし、`Approve and deploy` をクリックすると、デプロイが実行されます。

デプロイに関する情報は、以下のようにレポジトリのトップページの右側で確認できます。

| レビュー待ち | デプロイ中 | デプロイ後 |
| ---------- | --------- | -------- |
| ![Waiting](/images/approve-deployments-with-github-environments/waiting.png) | ![In progress](/images/approve-deployments-with-github-environments/in-progress.png) | ![Active](/images/approve-deployments-with-github-environments/active.png) |

後は、Actions のそれぞれの Summary ページで、レビューに関する情報を確認できます。

![Summary](/images/approve-deployments-with-github-environments/summary.png)

また、通知の設定によりますが、他のユーザーがデプロイのレビューを依頼した場合、Web 版やモバイルアプリにも通知が来ます。

# 最後に

GitHub Environment を活用すると簡単に承認機能が付けられて便利です。

私の場合は、名前的に承認機能とは思わず、しばらくスルーしてしまっていました。

多くの方にとって便利な機能だと思うので、是非活用してみてください。

こちらの記事が参考になれば幸いです。
