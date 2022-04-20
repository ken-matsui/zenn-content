---
title: "GitHub Actions の if と env"
emoji: "📜"
type: "tech"
topics: ["github", "githubactions"]
published: true
---

GitHub Actions に関して、どこに `env` が書けるんですか？とか、`if` ってどうやって使うんですか？とか色々と質問を受けることがあります。

そういった、`if` と `env` に関してフォーカスして、こちらの記事で紹介させていただきます。

# `env`

https://docs.github.com/en/actions/learn-github-actions/environment-variables

まず、`env` に関してです。

## `env` が使える場所

`env` の対象範囲は、`step` 単位です。

`env` が指定されていると、環境変数として、`step` 毎に展開されます。

```yaml
steps:
  - name: "Say Hello, world!"
    run: echo "$HELLO, world!"
    env:
      HELLO: Hello
```

では例えば、全ての `step` にて同じ環境変数を使用したい時は、以下のように `job` 単位でも同一の環境変数を定義できます。

```yaml
echo_job:
  runs-on: ubuntu-latest
  env:
    HELLO: Hello
  steps:
    - name: "Say Hello, world!"
      run: echo "$HELLO, world!"
    - name: "Say Hello, foo!"
      run: echo "$HELLO, foo!"
    - name: "Say Hello, bar!"
      run: echo "$HELLO, bar!"
```

さらに、`job` を跨ぐ環境変数も定義できます。

```yaml
env:
  HELLO: Hello

jobs:
  echo_job:
    runs-on: ubuntu-latest
    steps:
      - name: "Say Hello, world!"
        run: echo "$HELLO, world!"

  hello_job:
    runs-on: ubuntu-latest
    steps:
      - name: "Say Hello, world!"
        run: echo "$HELLO, world!"
```

ファイル全体での環境変数定義は非常に便利ですが、1 つの `step` でしか使用しない環境変数をそのように定義してしまうのは良くないので避ける方が望ましいです。

あくまで、対象範囲を広げざるを得なくなった時に、`job` 単位やファイル単位に広げるようにしましょう。

## `job` 内で環境変数をセットする

https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#setting-an-environment-variable

`job` の実行結果に応じて、変数が出力され、それを後から環境変数として使用したい時があります。

その時は、`GITHUB_ENV` に対して、`key=value` の形式で追記すると、その後でも環境変数としてアクセスできます。

```yaml
steps:
  - name: "Output docker image url"
    run: echo 'DOCKER_IMAGE_URL=gcr.io/cloudrun/hello' >> $GITHUB_ENV

  - name: "Show the docker image url"
    run: echo "$DOCKER_IMAGE_URL"
```

## `env` を contexts として使用する

https://docs.github.com/en/actions/learn-github-actions/contexts

`${{ github.sha }}` でそのコミットハッシュが取れたりするのですが、その `github.sha` の部分が contexts にあたります。

例えば以下のケースを考えてみます。

```yaml
steps:
  - name: "Output docker image url"
    run: echo 'DOCKER_IMAGE_URL=gcr.io/cloudrun/hello' >> $GITHUB_ENV

  - uses: 'google-github-actions/deploy-cloudrun@v0'
    with:
      service: 'hello-cloud-run'
      image: ''  # 環境変数が使えない？
```

この時、`with` として、環境変数を渡したいわけです。

例えば以下のようにしても、そのままの内容が入ってしまうため、エラーになってしまいます。

```yaml
      image: '$DOCKER_IMAGE_URL'
```

そこで、contexts を使用すると、簡単に中身を展開できます。

```yaml
      image: '${{ env.DOCKER_IMAGE_URL }}'
```

contexts はどんなところでも展開してくれるので、`run` の中に書いても大丈夫です。

```yaml
steps:
  - name: "Show the docker image url"
    run: echo '${{ env.DOCKER_IMAGE_URL }}'
```

違いは、bash が実行時に中身を展開するか、GitHub Actions が実行前に展開するかの違いです。

# `if`

次は `if` に関してです。

## `if` が使える場所

https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions

`if` は `env` と違って、ファイル単位では指定できません。

指定できるのは、以下のように `job` 単位と `step` 単位のみです。

```yaml: job 単位
jobs:
  assign:
    if: github.actor != 'dependabot[bot]'
```

```yaml: step 単位
jobs:
  assign:
    runs-on: ubuntu-latest
    steps:
      - name: Notify actor
        if: github.actor != 'dependabot[bot]'
        run: bash ./notify
```

## `if` はカッコを省略できる

https://docs.github.com/en/actions/learn-github-actions/expressions#about-expressions

`if` における `${{ }}` は、expression syntax と言うみたいですが、それは省略できます。

例えば、`dependabot` がワークフローを実行したユーザーではない時に、`job` を実行したいときには、以下のようにできます。

```yaml
jobs:
  assign:
    if: github.actor != 'dependabot[bot]'
```

省略できるだけなので、以下のように書いても問題ありません。

```yaml
jobs:
  assign:
    if: ${{ github.actor != 'dependabot[bot]' }}
```

## `if` と `env` を組み合わせる

https://docs.github.com/en/actions/learn-github-actions/expressions#startswith

`if` と `env` を組み合わせてみましょう。

以下のように使用できます。

```yaml
steps:
  - name: "Output docker image url"
    run: echo 'DOCKER_IMAGE_URL=gcr.io/cloudrun/hello' >> $GITHUB_ENV

  - uses: 'google-github-actions/deploy-cloudrun@v0'
    if: startsWith(env.DOCKER_IMAGE_URL, 'gcr.io')
    with:
      service: 'hello-cloud-run'
      image: '${{ env.DOCKER_IMAGE_URL }}'
```

## `if` & `env` & `startsWith` & `!`

色々と組み合わせてみましょう。

```yaml
- uses: 'google-github-actions/deploy-cloudrun@v0'
  if: ${{ !startsWith(env.DOCKER_IMAGE_URL, 'gcr.io') }}
  with:
    service: 'hello-cloud-run'
    image: '${{ env.DOCKER_IMAGE_URL }}'
```

`!` (否定演算子) と合わせて使用する時は、上記のように、expression syntax が必須になります。

こういった話は、以下に書かれています。

https://github.community/t/if-not-startswith-mutually-exclusive-steps/141841/2

# 最後に

色々と細かいところで、迷うところがあると思います。

特に expression に関しては、syntax が少し特殊なので、慣れるまで少し時間がかかる印象です。

また、以下の記事で紹介させていただいた `actionlint` を使用すれば、事前にそういった部分もチェックができます。

https://zenn.dev/matken/articles/5-github-actions-for-running-start

その場合は、GitHub Actions としてではなく、ローカルで `actionlint` を実行すると push する前にミスを発見できるので安心です。

こちらの記事が参考になれば幸いです。
