---
title: "モノレポでの無駄な GitHub Actions の実行を避ける"
emoji: "💨"
type: "tech"
topics: ["monorepo", "githubactions", "github"]
published: true
---

最近では多くの開発において、モノレポが採用されているかと思います。

その時、GitHub Actions はトップレベルでしか用意できないため、フロントエンドしか更新していないのに、バックエンドのテストが走ってしまいますが、あまり意味がありません。
むしろ、Organization の quota をなるべく消費したくないはずです。

そこで、関係無い変更が行われたときに、その CI を無視するように設定する方法について、こちらの記事で紹介させていただきます。

:::message
ただ単に、`job` 単位での `if` や `on` 等を使って GitHub Actions が実行されないようにする方法が一番簡単ですが、こちらで紹介する方法は Branch Protection Rules を有効にしていても正しく処理するための方法となります。
:::

# `paths-filter` を使用する

https://github.com/dorny/paths-filter

`paths-filter` を使用すると、特定のファイル変更に対して、GitHub Actions の `if` で使用するための output が提供されます。

こちらをモノレポで使っていくには、以下のように設定していきます。

`.github/filters.yml` を用意します。
こちらは、YAML の anchor と alias を使用できるため、以下のように書けます。

```yaml: .github/filters.yml
backend_ci: &backend_ci
  - '.github/workflows/backend*.yml'

backend:
  - *backend_ci
  - 'backend/**'


frontend_ci: &frontend_ci
  - '.github/workflows/frontend*.yml'

frontend:
  - *frontend_ci
  - 'frontend/**'
```

そうすると、以下のように、フロントエンドの変更があった時だけ Lint を実行する GitHub Actions が書けます。

```yaml: .github/frontend-lint.yml
- uses: actions/checkout@v3.0.0

- name: Check for file changes
  uses: dorny/paths-filter@v2
  id: changes
  with:
    token: ${{ github.token }}
    filters: .github/filters.yml

- if: steps.changes.outputs.frontend == 'true'
  run: npm run lint
```

Branch Protection Rules を通すには、成功したように見せかける必要があります。

そのため、`job` 単位では実行する必要があり、`step` 単位ではスキップしていくという流れになります。

上記の例では、ファイルの変更をチェックした後、一度のみ step を実行していますが、その後も step がある場合は、全てに `if` を付けていく必要があります。
そうすることで、`frontend/**` に対して変更が無い場合は、ほぼ一瞬で GitHub Actions が成功するようになります。

`paths-filter` の設定ファイルを読む必要があるため、`job` に対して `permissions` を明示的に書きたい場合は、以下のようにする必要があります。

```yaml
jobs:
  lint:
    permissions:
      contents: 'read'
      pull-requests: 'read'
```

# Branch Protection Rules に指定されていない job

`main` ブランチにマージされたら、開発環境のデプロイを行う Action の場合は、Branch Protection Rules に入っていないはずです。

そういったものは、単純に以下のように `job` ごとスキップするように実装すると、`step` 毎に `if` を書く必要が無く、綺麗です。

```yaml
jobs:
  changes:
    permissions:
      contents: 'read'
      pull-requests: 'read'

    runs-on: ubuntu-latest

    outputs:
      backend: ${{ steps.filter.outputs.backend }}
    steps:
      - uses: actions/checkout@v3.0.0
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          token: ${{ github.token }}
          filters: .github/filters.yml

  deploy:
    permissions:
      contents: 'read'

    needs: changes
    if: needs.changes.outputs.backend == 'true'

    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3.0.0
        # if: は不要

        ...
```

# 最後に

どうしても、`step` 毎に `if` を書くと汚く見えてしまいます。
また、他の条件も組み合わせたい場合は、`&&` で繋げていく必要があります。

最初はあまり納得がいかず、色々と調べてみましたが、現状、Branch Protection Rules と組み合わせたい場合はこの方法しか無いみたいです。

こちらの記事が参考になれば幸いです。
