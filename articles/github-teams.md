---
title: "GitHub Teams を会社として使う"
emoji: "👨‍👩‍👧‍👦"
type: "idea"
topics: ["github"]
published: true
---

GitHub Teams を活用すると、それぞれのレポジトリに対する権限を一元管理でき、非常に便利です。
しかし、今まで関わってきた会社の中には GitHub Teams が使用されていても、ただのラベル付けのようにしか使用されておらず、権限管理には使用されていないところもありました。

そこで今回は、GitHub Teams を会社として運用するにはどうすれば良いのか、に関して紹介させていただきます。

# Organization でのロールに関して

まず、Organization でのロールは 2 種類存在します。
`Owner` と `Member` のみで、`Owner` はその Organization に対して何でもできる権限で、`Member` は、後述する `Member privileges` や Teams によってできることが変化します。

![Roles](/images/github-teams/roles.png)

基本的には、インハウスのエンジニアリーダー数人が `Owner` でそれ以外は、`Member` という運用になるかと思います。

業務委託のエンジニアをどう扱うかに関して、よく迷うことがあると思いますが、私は `Member` で招待しています。
理由は、Outside Collaborator が Teams と組み合わせることができないからで、プロジェクトが複数レポジトリに跨ぐ場合に、それぞれに権限を付与していくことが大変だからです。
また、契約満了時に、権限を外していくのも面倒です。
さらに、どちらにせよ、メンバーの Seat 数を消費するため、Outside Collaborator であっても課金が発生します。

そのため、`Member` として招待し、Teams で柔軟に権限を管理する、という運用がおすすめです。

# Member privileges

まずは、`Member privileges` を確認しましょう。

Organization のページから、`Settings` > `Member privileges` へ移動します。

## Base permissions

`Base permissions` 欄を確認すると、デフォルトであれば `Read` となっています。

しかし、会社においては、メンバーに招待はしていても、完全にインハウスでは無いエンジニアの場合もあります。
そういった場合、関わってもらうプロジェクト以外のプロジェクトのレポジトリまで、全て見えてしまうことになります。
`Read` が付与されているということなので、Clone が可能です。

私が過去関わった会社で、業務委託のエンジニアがプロジェクトのソースコードを個人の GitHub アカウントの方で `Public` として公開していたことがありました。

あえて Fork をしていないことで、こちらとしては気付きづらく、指摘したところで `Private` に変更しているだけかもしれません。
つい最近 Heroku と Travis CI の OAuth トークンが流出した事件がありましたが、`Private` にしていたとしてもソースコードが流出する可能性があります。

https://zenn.dev/hiroga/articles/heroku-incident-2413-checklist

つまり、レポジトリが見えているということだけで、問題になり得ます。
なので、会社として GitHub Organization を運用する場合は、`Base permissions` を、`No permission` に設定することをおすすめします。

`No permission` に設定することで、その Organization に `Member` として参加した場合は、その Organization に参加していないユーザーと同様に `Public` レポジトリしか見ることができません。

## Repository creation

もし、Enterprise として契約している場合は、`Member` が `Public` レポジトリを作成することを制限できます。

# Admin repository permissions

## Repository visibility change

> Allow members to change repository visibilities for this organization

上記の設定がデフォルトではオンになっていますが、せっかくレポジトリを `Private` にしていても `Public` に変更されたら意味がありません。
こういったことは、エンジニアのリーダー的な存在が担当すべきなので、オフにしておくことをおすすめします。

## Repository deletion and transfer

> Allow members to delete or transfer repositories for this organization

上記の設定もデフォルトではオンになっています。

こちらも同様で、勝手に削除されたり、個人のアカウントにレポジトリを移されたら問題です。
こちらに関しても、エンジニアリーダー的なポジションの人のみが担当すべきなので、オフにしておくと良いでしょう。

# Member team permissions

## Team creation rules

> Allow members to create teams

上記の設定に関してもデフォルトでオンになっています。

こちらもエンジニアリーダーが管理した方が良いので、オフにしておくと良いかと思います。

# Team を作成する

次に Teams を作成してみます。

Organization のページから、Teams へ移動してください。

そして、`New team` というボタンをクリックし、例えば Example プロジェクトであれば、`Example PJ` と入力します。

![Team Creation](/images/github-teams/team-creation.png)

入力できたら、`Create team` ボタンをクリックします。

# Team にレポジトリに対する権限を設定する

まずは、そのプロジェクトに関わるレポジトリ達を、`Read` 権限で追加してみます。

`Add repository` をクリックして、レポジトリを選択し、`Add repository to team` をクリックします。

![Add repo](/images/github-teams/add-repo.png)

すると、以下のように `Read` 権限でそのレポジトリが追加されたことが確認できます。

![Added repo](/images/github-teams/added-repo.png)

# Team に Member を招待する

次に、`Members` に移動します。

そこで、`Add a member` というボタンを押すと、メンバーを追加できます。
この時、組織にまだ所属していないメンバーでも追加できます。

なので、組織に招待してから、Team に入れるという手間が省けます。

招待されたユーザーは、先程追加したレポジトリのみが見えるようになっています。

# Teams の権限の仕組み

まず、`Base permission` という基盤があり、その上に、Teams による権限によって上書きされていきます。

そして、複数の Team で権限が重なっている場合は、権限の強い方が優先されます。

* `Example PJ`: `test` レポジトリに `Read` 権限
* `Test PJ`: `test` レポジトリに `Write` 権限

といった Team が存在していて、その両方の Team に所属している場合は、`test` レポジトリに対して `Write` 権限を持っています。

# Teams を入れ子にする

また、Teams は入れ子にできます。
例えば、以下のように `Example PJ Engineers` Team を、 `Example PJ` を親として作成してみましょう。

![Team Creation 2](/images/github-teams/team-creation2.png)

すると、`Team Members` は引き継がれず、`Repository` は引き継がれていることが確認できます。
また、`Repository` では、権限が引き継がれています。
そして、「Teams の権限の仕組み」節で説明したのと同様に、その権限は、さらに強い権限で上書きできます。

![Permissions](/images/github-teams/permissions.png)

エンジニアは、`Write` 以上の権限が無いと開発に参加できないので、`Write` に変更してみましょう。
そしてメンバーを追加してみます。

すると、`Example PJ Engineers` Team に追加されたユーザーは、`push` 等ができるようになりました。

# 最後に

以上で、全体的な Teams の使い方が説明できたかと思います。

実際に先程作成した Teams を使い分けるとすれば、例えば、Biz メンバーも GitHub に参加しているのであれば、`Example PJ` Team の権限で充分です。
そして、実際の開発に参加するエンジニアは、`Example PJ Engineers` Team に参加するという感じになります。

こちらの記事では、プロジェクト毎で Team を分割し、その中でチーム毎に分割していますが、その他の切り方もあると思います。
会社によって色々な切り方があると思うので、色々と試してみてください。

また、以下の記事に Teams の使い方が詳細に書かれているので、参考にしてください。

https://docs.github.com/en/organizations/organizing-members-into-teams/about-teams

こちらの記事が参考になれば幸いです。
