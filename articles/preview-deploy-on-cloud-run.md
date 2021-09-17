---
title: "Cloud Runを使ってPR毎にプレビューデプロイを行う"
emoji: "🚀"
type: "tech"
topics: ["gcp", "cloudrun", "github", "githubactions"]
published: true
---

GitHub での複数人開発において、レビュー機能を使用している場合、プルリクエスト毎にレビュー用の環境があれば、コードのみレビューに加えて目視でのレビューが行えます。特に、フロントエンドコードであれば、コード上では確認しきるのが難しい部分も多く、目視でのレビューは非常に大切です。そして、その環境がプルリクエスト毎に別れていれば、レビュー範囲をレビュアーが手動で切り替える、レビューするタイミングで毎回デプロイを切り替える等の作業が必要なくなります。

Firebase Hosting を使用している場合、GitHub Actions と連携することで、Preview Channel を使用して GitHub のプルリクエスト上にプレビューに対するリンクのコメント発行まで行ってくれます。そのため、レビュアーはそのコメントされたリンクをクリックするだけで、その PR 内の変更点のみを含んだ環境を閲覧・レビューできます。

![Firebase Hostingのプレビューリンクコメント](/images/git-workflow-in-waterfall-for-starups/hosting-github-action-previewURL-comment.png)
*Firebase Hosting のプレビューリンクコメント
Source: [Deploy to live & preview channels via GitHub pull requests](https://firebase.google.com/docs/hosting/github-integration)*

しかし、Firebase Hosting は SPA である必要があり、SSR にしたい場合やフロントエンドコードでない場合は、Firebase Hosting を使用できません。そういった時、GCP の Cloud Run を使用することで、複雑な設定無しに Firebase Hosting と同様のことを実現できます。

# PR毎にプレビューを自動生成する

Cloud Run には、Revision URL というものが用意されており、それが、Firebase Hosting における Preview Channel となっています。以下のリンクには、プルリクエスト毎に Revision URL を発行する方法が記述されています。

https://cloud.google.com/run/docs/tutorials/configure-deployment-previews

しかし、上記の記事では、GitHub の Personal Access Token や Cloud Build が必要になってしまいます。そちらで問題無い場合は、上記リンクのまま進めていただけます。ただ、大概の場合は、GitHub 上でのみ実現できると嬉しいし、Firebase Hosting みたいにコメント機能があれば尚嬉しいと思います。また、GitHub 上でのみ実現する場合、PAT 等も不要なため、構成としてとても単純に実現できます。

まずは GitHub Actions を使用して、プレビューデプロイを作成します。ワークフローファイルは以下になります。

```yaml:cloud-run.preview.yml
name: Cloud Run (Preview)

on: pull_request

jobs:
  deploy:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        service: [ product-x ]

    steps:
      - uses: actions/checkout@v2.3.4

      - name: Create docker image url
        id: image-url
        run: echo "::set-output name=value::gcr.io/${{ secrets.DEV_GCP_PROJECT_ID }}/${{ matrix.service }}-service:${{ github.event.pull_request.number }}-${{ github.event.pull_request.head.sha }}"

      - name: Build image
        run: docker build -t ${{ steps.image-url.outputs.value }} -f ./docker/development/Dockerfile .

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@master
        with:
          project_id: ${{ secrets.DEV_GCP_PROJECT_ID }}
          service_account_key: ${{ secrets.DEV_GCP_SA_KEY }}
          export_default_credentials: true

      - name: Configure docker to use the gcloud cli
        run: gcloud auth configure-docker --quiet

      - name: Push image
        run: docker push ${{ steps.image-url.outputs.value }}

      - name: Install beta component
        run: gcloud components install --quiet beta

      - name: Deploy revision with tag
        run: >
          gcloud beta run deploy ${{ matrix.service }}
          --platform managed
          --region ${{ secrets.GCP_REGION }}
          --image ${{ steps.image-url.outputs.value }}
          --tag pr-${{ github.event.pull_request.number }}
          --no-traffic
```

初めに、Docker Image のプッシュ先 URL が非常に長くなってしまうため、そちらをステップの output として設定しています。Container Registry として、GCR を設定しています。Docker Image のビルドステップでは、対象の Dockerfile のパスは適宜変更してください。

私が確認した時点では、beta コンポーネントのインストールが必要だったのですが、現状は不要になっている可能性があります。もし不要になっている場合は、コメントいただけると嬉しいです。
その beta コンポーネントを使用して、`--tag` 引数部分で、PR 用のプレビューリンクの prefix を指定しています。今回だと、`pr-${PR_NUMBER}` というフォーマットにしています。

Cloud Run 自体、独自のドメインを持っていると思いますが、そちらに先程指定したタグを使用して `https://${tag}---${domain}` という形で、PR 用のプレビューリンクがフォーマットされます。

一部変則的な部分を除き、基本的にシンプルかなと思います。GCP のプロジェクトと、サービスアカウントキーを用意し、以下の表のようにシークレットを登録してください。

| シークレット名       | 例               |
| ------------------ | --------------- |
| GCP_REGION         | asia-northeast1 |
| DEV_GCP_PROJECT_ID | product-x-dev   |
| DEV_GCP_SA_KEY     | `cat sa.json | base64 | pbcopy` |

新規 PR を発行すれば、正常に実行されるかどうか確認できます。正常に実行されれば、Cloud Run のコンソールにて、以下の画像のようにタグが切られています。タグである、`pr-` の後ろの番号は、新規発行したプルリクエストの番号になっているはずです。

![Cloud RunのRevision URL](/images/cloud-run-revision-url-per-pr/cloud-run-revision-url.png)
*Cloud Run の Revision URL*

また、コンソール上のタグをクリックすることで、今回の PR 内で行った変更点のみが適用されていることが確認できます。

![Cloud Runのプレビュー](/images/cloud-run-revision-url-per-pr/preview.png)
*Cloud Run のプレビュー*

# プレビューリンクをPR上にコメントする

Cloud Run に自動でデプロイされても、リンクを毎回手動で入力するのは非常に面倒です。Firebase Hosting のように自動でプレビューリンクをコメントしてくれると、レビューが行いやすくなります。先程作成した、`cloud-run.preview.yml` ファイルに対して、以下の変更を末尾から付け加えてください。

```yaml:cloud-run.preview.yml
      - name: Find Comment
        uses: peter-evans/find-comment@v1
        id: fc
        with:
          issue-number: ${{ github.event.pull_request.number }}
          comment-author: github-actions[bot]
          body-includes: "Preview"

      - name: Create Preview URL
        id: preview-url
        run: echo "::set-output name=value::https://pr-${{ github.event.pull_request.number }}---product-x-drwvbiotlz-an.a.run.app"

      - name: Get datetime for now
        id: datetime
        run: echo "::set-output name=value::$(date)"
        env:
          TZ: Asia/Tokyo

      - name: Create or update comment
        uses: peter-evans/create-or-update-comment@v1
        with:
          comment-id: ${{ steps.fc.outputs.comment-id }}
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            Visit the :eyes: **Preview** :eyes: for this PR (updated for commit ${{ github.event.pull_request.head.sha }}):
            <${{ steps.preview-url.outputs.value }}>
            <sub>(:fire: updated at ${{ steps.datetime.outputs.value }})</sub>
          edit-mode: replace
```

`Create Preview URL` ステップのみ、URL 部分を Cloud Run が自動生成するドメインに適宜変更してください。

細かな挙動の説明は省きますが、上記の変更を加えれば、下の画像のように、PR 上にコメントが追加されるかと思います。

![Cloud Runのプレビューリンクコメント](/images/cloud-run-revision-url-per-pr/preview-url-comment.png)
*Cloud Run のプレビューリンクコメント*

# PRがクローズされた時、プレビューリンクを削除する

プレビューリンクは基本的に、社外から直接アクセス可能になってしまいます。タグを使用したリンクの場合は、Cloud Run が自動生成したリンクになるため、Firebase Hosting も同様ですが、アクセス保護することが難しいです。

そこで、PR が開かれている間だけアクセス可能な状態であれば、実質的に社外からはアクセス不可能とみなすことができるかと思います。もちろん、プレビューリンクのタグに PR 番号以外のランダムな文字列を付け加えれば、さらにセキュアにできます。

また、会社のレギュレーション的に絶対的な保護が必要だ、という場合は、ステージング環境時にのみ Basic 認証等をかける対応策が考えられます。

どちらにせよ、PR がマージもしくはクローズされた時、レビューのために用意していたプレビューリンクは不要になります。Cloud Run 上に不要なアクセス可能なタグがあるのはあまり良いものでないので、GitHub Actions を用いて、そちらを自動で削除するようにします。

```yaml:cloud-run.delete-preview.yml
name: Cloud Run (Delete Preview)

on:
  pull_request:
    types: [ closed ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        service: [ product-x ]

    steps:
      - uses: actions/checkout@v2.3.4

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@master
        with:
          project_id: ${{ secrets.DEV_GCP_PROJECT_ID }}
          service_account_key: ${{ secrets.DEV_GCP_SA_KEY }}
          export_default_credentials: true

      - name: Install alpha component
        run: gcloud components install --quiet alpha

      - name: Deploy revision with tag
        run: >
          gcloud alpha run services update-traffic ${{ matrix.service }}
          --region ${{ secrets.GCP_REGION }}
          --remove-tags pr-${{ github.event.pull_request.number }}
```

上記の GitHub Actions ファイルを用意し、PR をクローズすれば、タグが削除されたことを確認できます。

![Cloud Runの削除されたRevision URL](/images/cloud-run-revision-url-per-pr/cloud-run-deleted-revision-url.png)
*Cloud Run の削除された Revision URL*

また、PR 上のコメントも更新したくなるかもしれません。そういった時は、以下のコードを `cloud-run.delete-preview.yml` の末尾に付け加えてみてください。

```yaml:cloud-run.delete-preview.yml
      - name: Find Comment
        uses: peter-evans/find-comment@v1
        id: fc
        with:
          issue-number: ${{ github.event.pull_request.number }}
          comment-author: github-actions[bot]
          body-includes: "Preview"

      - name: Create Preview URL
        id: preview-url
        run: echo "::set-output name=value::https://pr-${{ github.event.pull_request.number }}---product-x-drwvbiotlz-an.a.run.app"

      - name: Get datetime for now
        id: datetime
        run: echo "::set-output name=value::$(date)"
        env:
          TZ: Asia/Tokyo

      - name: Create or update comment
        uses: peter-evans/create-or-update-comment@v1
        with:
          comment-id: ${{ steps.fc.outputs.comment-id }}
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            Visit the :eyes: **Preview** :eyes: for this PR (updated for commit ${{ github.event.pull_request.head.sha }}):
            ~<${{ steps.preview-url.outputs.value }}>~
            <sub>(:warning: deleted at ${{ steps.datetime.outputs.value }})</sub>
          edit-mode: replace
```

上記の変更を適用すると、以下の画像のように、コメントが更新されるようになります。

![Cloud Runの削除されたプレビューリンクのコメント](/images/cloud-run-revision-url-per-pr/deleted-preview-url-comment.png)
*Cloud Run の削除されたプレビューリンクのコメント*

試しにプレビューリンクへ再度アクセスすると、404 が返ってくることが確認できます。

![Cloud Runの削除されたプレビュー](/images/cloud-run-revision-url-per-pr/deleted-preview.png)
*Cloud Run の削除されたプレビュー*

ある程度手間がかかるものの、一度整えてしまえば長期的に多大なレビューコストを低減させてくれます。実際にこちらを社内に導入してから 3 ヶ月以上経ちますが、非常に安定しています。Firebase Hosting と違い、SPA に縛られず、バックエンドコード等にも使用できる点が非常に便利な点かなと思います。
