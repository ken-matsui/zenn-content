---
title: "共同開発を始めるときに便利な 5 つの GitHub Actions"
emoji: "📑"
type: "idea"
topics: ["github", "githubactions"]
published: true
---

# はじめに

スタートアップ等において新しいプロダクトを始める時は、負債が無い代わりに何もありません。
そういった時に、ソフトウェアの品質を担保するための CI のセットアップが、初期から重要になってきます。

GitHub を使用している場合は、GitHub Actions を使用されることが殆どだと思うので、そちらを前提に進めていきたいと思います。

# 1. rhysd/actionlint

https://github.com/rhysd/actionlint

様々なエンジニアが action を追加したり、編集したりするようになった時、全員が正しい書き方で書いていくことは難しいです。
また、それを 1 人の GitHub Actions Expert がレビューしていくのは大変で、属人化してしまっているので、避ける方が望ましいです。

---

以下をコピペすれば、使用できます。

`.github/workflows/actionlint.yml`

```yaml
name: Actionlint

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  actionlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Download actionlint
        id: get_actionlint
        run: bash <(curl https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash)
        shell: bash

      - name: Check workflow files
        run: ${{ steps.get_actionlint.outputs.executable }} -color
        shell: bash
```

# 2. technote-space/assign-author

https://github.com/technote-space/assign-author

こちらを導入することで、PR の Assignee として PR 作成者を Assign できます。
こちらの利点としては、PR の作成者、つまりそのタスクの担当者が、PR 一覧にて一目で分かることです。

![Assign](/images/5-github-actions-for-running-start/assign.png)

例えば、B さんの PR かなり前から Draft のままだから、開発詰まってるのかな〜とかが判断できます。

---

以下をコピペすれば、使用できます。

`.github/workflows/auto-assign-author.yml`

```yaml
name: Auto Assign Author

on:
  pull_request:
    types: [ opened ]

jobs:
  assign:
    if: github.actor != 'dependabot[bot]'

    name: Assign author to PR
    runs-on: ubuntu-latest
    steps:
      - name: Assign author to PR
        uses: technote-space/assign-author@v1
```

# 3. actions/labeler

https://github.com/actions/labeler

モノレポで運用している時、どの PR がどのマイクロサービスに対応するのかが、PR 一覧では分かりづらいことが多いです。
そういった時に、その PR の変更点を元に、ラベルを付与させることができます。

![Label](/images/5-github-actions-for-running-start/label.png)

---

以下をコピペすれば、使用できます。

`.github/workflows/labeler.yml`

```yaml
name: "Pull Request Labeler"
on:
  - pull_request_target

jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/labeler@v4
      with:
        repo-token: "${{ secrets.GITHUB_TOKEN }}"
        sync-labels: true
```

`.github/labeler.yml`

```yaml
Service-frontend: 'frontend/**/*'
Service-backend: 'backend/**/*'
```

# 4. ken-matsui/notify-slack

https://github.com/ken-matsui/notify-slack

こちらに関しては、Zenn の記事にもしているので、是非参考にしてください。

https://zenn.dev/matken/articles/notify-slack-action

GitHub で共同開発していると、どうしても連絡やタスクを見逃しやすいです。
そういったときに、Slack にタスク関係の連絡のみを通知するための action を開発してみました。

![Received Review](/images/notify-slack-action/received-review.png)

---

上記記事内のセットアップと、以下をコピペすれば、使用できます。

`.github/workflows/notify-slack.yml`

```yaml
name: GitHub Notification

on:
  pull_request:
    types: [review_requested, closed]
  pull_request_review:
    types: [submitted]
  issue_comment:
  pull_request_review_comment:

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - uses: ken-matsui/notify-slack@v1.0.0
        with:
          slack_oauth_access_token: ${{ secrets.SLACK_OAUTH_ACCESS_TOKEN }}
```

`.github/userlist.toml`

```toml
[[users]]
github = "ken-matsui"
slack = "UXXXXXXXXXX"

[[users]]
github = "ken-matsui-2"
slack = "UXXXXXXXXXX"
```

# 5. secretlint/secretlint

https://github.com/secretlint/secretlint

初期では、GitHub レポジトリは Private だと思いますが、今後、社内プロダクトを OSS にすることを考えている会社もあると思います。
また、業務委託等が開発に参加している場合にも、外部に対して、内部のシークレットキーを公開することはあまり望ましくありません。

エンジニアが誤ってシークレットキーを commit しちゃった時にも、いつからか公開されていた、ということが起きずに迅速にキーの再生成が可能です。

---

以下をコピペすれば、使用できます。

`.github/workflows/secretlint.yml`

```yaml
name: Secretlint

on:
  push:
    branches: [ main ]
  pull_request:

permissions:
  contents: read

jobs:
  secretlint:
    runs-on: ubuntu-latest
    steps:
      - name: checkout
        uses: actions/checkout@v3

      - name: setup Node.js
        uses: actions/setup-node@v3.1.1
        with:
          node-version: 17

      - name: Install dependencies
        run: yarn

      - name: Lint with Secretlint
        run: yarn secretlint
```

`package.json`

```json
{
  ...,
  "scripts": {
    "secretlint": "secretlint '**/*'"
  },
  "devDependencies": {
    "@secretlint/secretlint-rule-preset-recommend": "5.2.0",
    "secretlint": "5.2.0"
  }
}
```

# 最後に

初期のレポジトリ整備は、非常に大変なことですが、今後に関わる重要なことです。
こういった GitHub Actions を最大限活用して、開発をブーストできると良いかなと思います。

また、この記事が参考になれば幸いです。
