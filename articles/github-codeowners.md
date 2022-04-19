---
title: "GitHub で Code Owners を会社として活用する"
emoji: "🔏"
type: "idea"
topics: ["github"]
published: true
---

GitHub には Code Owners という機能があります。

https://docs.github.com/ja/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners

この機能は、`CODEOWNERS` ファイルを `.github/` ディレクトリ内に置いておくことで、特定のディレクトリやファイルを編集した PR を作成した時に、自動でその編集箇所の Owner に対してレビューリクエストを送信するための機能です。

例えば、以下のようにモノレポ構成時に、`frontend` と `backend` それぞれに対して詳しいチームリーダーがいたとすると、それぞれの変更箇所に応じて PR 作成者がレビュアーを探すことなくレビューリクエストを送信できます。

```: .github/CODEOWNERS
frontend/*  @frontend-leader-f-san
backend/*   @backend-leader-b-san
```

こういった機能は、不特定多数が Contribute する環境である、OSS で役立てられているんじゃないかなと思います。

この程度であれば、そもそもスクラムチームをそこまで大きくしないため、誰にレビューリクエストを出せば良いという情報はキチンと共有されていると思いますし、そこまで利点には感じません。

しかし、実は Code Owners は Owner として [Teams](https://docs.github.com/en/organizations/organizing-members-into-teams/about-teams) を指定でき、さらに、[Branch Protection Rule](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/managing-a-branch-protection-rule) と組み合わせることができます。

これらが非常に便利なため、こちらの記事で紹介させていただければと思います。

# Code Owners と Branch Protection Rule

特に会社として運用している場合は、`main` ブランチを Branch Protection Rule を使用して保護していると思います。

その時、以下のように Branch Protection Rule を構成できます。

![Branch Protection Rule](/images/github-codeowners/branch-protection-rule.png)

上記のように、`Require 1 approval` と `Require review from Code Owners` を組み合わせることで、先程の例を参照すると、`@frontend-leader-f-san` か `@backend-leader-b-san` のどちらかのレビューを通過しないと `main` ブランチにマージが行えないようになります。

GitHub Actions 等で、`main` ブランチを起点に自動デプロイ等を行っている場合、どうしてもそのレポジトリに対して `Write` 権限を持った全てのエンジニアがデプロイを行えてしまうため、意図しないリリースが発生してしまいます。

そういう時に、ソースコードの品質を担保する役割のユーザーを Code Owners に指定することで、安全にサービスを運用できるようになります。

# Code Owners と Teams

例えば、以下のような Teams 構造の時を考えてみます。

* `example` レポジトリ
* `Example PJ` Team: `example` レポジトリに対する `Read` 権限
  * `Example PJ Engineers` Team: `example` レポジトリに対する `Write` 権限 (push するため)
  * `Example PJ Codeowners` Team: `example` レポジトリに対する `Write` 権限 (レビューするため)

この時、`example` レポジトリのソースコード品質を担保する役割のユーザーを、`Example Codeowners` Team に招待し、それ以外のエンジニアを、`Example Engineers` Team へ招待します。

すると、CODEOWNERS ファイルは以下のように記述できます。

```: .github/CODEOWNERS
*       @example-inc/example-pj-codeowners
```

そして、`Example Codeowners` Team のページへ移動し、Settings > Code review と遷移します。

すると以下のようなページになっていると思います。

![Code Review Page](/images/github-codeowners/code-review.png)

ここで、`Enable auto assignment` を有効にしておくと、こちらの Team を経由して自動でレビューリクエストが飛びます。

また、複数人いたとしても、いい感じのバランスを見て GitHub がレビューリクエストを投げてくれます。

上記の設定をしていると以下のように、PR が Open された or Draft から Ready for review にされたタイミングで、Team を経由して自動でレビューが飛ぶようになります。

![Code Owner review request](/images/github-codeowners/codeowner-review.png)

そしてそのレビューは、Code Owners の権限の元で行われるため、上記の Branch Protection Rule を設定していると、マージできるようになります。

![Code Owner approve](/images/github-codeowners/codeowner-approve.png)

# 最後に

Codeowners は最高レベルでレポジトリを保護することに繋がりますが、逆にプロジェクトの進みを鈍化させることもあります。
自分が倒れてしまった場合でも上手くプロジェクトが回るように、複数人を Codeowners にしておく等、何かしらの対応を施しておくと安心かと思います。

また、Team の Code Review ページは色々な設定があるため、是非色々と試してみてください。

こちらの記事が参考になれば幸いです。
