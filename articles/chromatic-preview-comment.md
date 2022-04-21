---
title: "Chromatic のプレビューリンクを自動で PR 上にコメントする"
emoji: "👀"
type: "tech"
topics: ["chromatic", "storybook", "github", "githubactions"]
published: true
---

Storybook を使用してコンポーネントを管理されている方は、Chromatic も合わせて使っているんじゃないかと思います。

以下の記事には、GitHub Actions を使用して Chromatic をデプロイする方法に関して記載されており、それで運用されている方が多いかと思います。

https://www.chromatic.com/docs/github-actions

しかし、GitHub を中心に活動している自分にとって、レビューのタイミングで毎度 Chromatic へログインして該当のデプロイを探すのは非常に面倒です。

該当のデプロイへ直接アクセスしたいものです。

そういった経緯から、以下のようにプレビューリンクの付いたコメントを生成してほしいと思っていました。

![Firebase Hosting のプレビューリンクコメント](/images/git-workflow-in-waterfall-for-starups/hosting-github-action-previewURL-comment.png)
*Firebase Hosting のプレビューリンクコメント
Source: [Deploy to live & preview channels via GitHub pull requests](https://firebase.google.com/docs/hosting/github-integration)*

Cloud Run では同様のことを実現していたため、そちらと同じような流れで実装していこうと思います。

https://zenn.dev/matken/articles/preview-deploy-on-cloud-run

# GitHub Actions を更新する

Chromatic のセットアップや、上記の Chromatic 公式記事内の GitHub Actions セットアップは完了している前提で進めさせていただきます。

`chromaui/action` には、output が用意されており、そちらを利用することで、プレビューリンク付きコメントを生成できます。

https://github.com/chromaui/action#outputs

## 1. `id` を付与する

GitHub Actions では、`id` を付与することで、後の `step` からその `step` で出力された output を取得できます。

https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsid

```yaml diff: .github/workflows/chromatic.yml
      - name: Publish to Chromatic
        uses: chromaui/action@v1
+        id: chromatic
        # Chromatic GitHub Action options
        with:
          # 👇 Chromatic projectToken, refer to the manage page to obtain it.
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
```

## 2. Chromatic の URL から不要なパスを削除する

上記の `Publish` 以降に繋げる形で追記してください。

出力される `storybookUrl` には不要な `iframe.html` が追加されており、それを削除して再度 output として出力します。

```yaml: .github/workflows/chromatic.yml
      - name: Remove unnecessary path for Chromatic link
        id: storybook-url
        run: echo "::set-output name=value::${STORYBOOK_URL//\/iframe.html/}"
        env:
          STORYBOOK_URL: ${{ steps.chromatic.outputs.storybookUrl }}
```

## 3. コメントを作成する

`find-comment` と `create-or-update-comment` を併用して、既存のコメントがなければ新規作成し、あればそれを更新するという処理を実装します。

`secrets.PAT_TOKEN` は、以下のリンクを参考に作成してください。
状況に応じて不要になるかもしれません。

https://github.com/peter-evans/create-or-update-comment#action-inputs

また、`create-or-update-comment` に渡す PAT の作成者が、そのままコメントするユーザーになります。
そのため、`find-comment` での `comment-author` には、その PAT に使用したユーザーの ID を入力してください。

```yaml: .github/workflows/chromatic.yml
      - name: Find Comment
        uses: peter-evans/find-comment@v2
        id: fc
        with:
          issue-number: ${{ github.event.pull_request.number }}
          comment-author: github-user-id
          body-includes: ':books: Storybook :books:'

      - name: Get datetime for now
        id: datetime
        run: echo "::set-output name=value::$(date)"
        env:
          TZ: Asia/Tokyo

      - name: Create or update comment
        uses: peter-evans/create-or-update-comment@v2
        with:
          token: ${{ secrets.PAT_TOKEN }}
          comment-id: ${{ steps.fc.outputs.comment-id }}
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            Visit the :books: **Storybook** :books: for this PR (updated for commit ${{ github.event.pull_request.head.sha }}):
            <${{ steps.storybook-url.outputs.value }}>
            <sub>Build URL: ${{ steps.chromatic.outputs.buildUrl }}</sub>
            <sub>(:fire: updated at ${{ steps.datetime.outputs.value }})</sub>
          edit-mode: replace
```

# 実行する

上手く設定できていれば、以下のように PR 上にコメントが作成されます。

PR 上のコミット毎に走ってしまいますが、その時は、コメントが重複しないように更新されます。

![Comment](/images/chromatic-preview-comment/comment.png)

# 最後に

以上で設定は完了です。

コメントの形式は自由に設定できるので試してみてください。

こちらの記事が参考になれば幸いです。
