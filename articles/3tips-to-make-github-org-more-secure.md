---
title: "GitHub Organizationをセキュアにするための3つのTips"
emoji: "🔐"
type: "idea"
topics: ["github", "security", "auth0", "googleworkspace", "saml"]
published: true
---

:::message
この記事は、以下のサービスに契約していることを前提にしています。
* GitHub Enterprise Cloud
* Google Workspace
:::

会社単位で GitHub Organization を運用する場合は、権限管理等を含めたセキュリティが非常に重要です。Github Free の Organization でも基本的な権限管理であれば揃っていますが、SAML 等の高度なセキュリティ設定は含まれていません。

そういった Free で可能なメンバーの権限管理に関しては、別の記事にて紹介します。こちらの記事では、Free や Enterprise にて利用可能で、重要な 3 つのセキュリティ設定に関して紹介します。

# Two-factor authentication を強制化する `(free)`

Two-factor authentication を組織として強制化していない場合、個人アカウントの乗っ取りによって、会社のソースコード等の流出の可能性があります。そういった危険性をなるべく排除するために、まずは個人アカウントを保護してもらうことが大切です。

公式ドキュメントがあるため、基本的にそちらを参考に進めていただければ問題ありません。

https://docs.github.com/en/organizations/keeping-your-organization-secure/requiring-two-factor-authentication-in-your-organization

以下の画像のように、ボタンをクリックするだけなので、簡単な設定かと思います。しかし、この時点で、2FA が有効でないユーザーは、強制的に組織から排除されてしまいます。さらに、組織に所属するメンバーと Outside Collaborator を含めた全員が影響するため、既存のメンバーが多くいる場合は少し大変です。その場合は、Slack 等で開発メンバー全体に周知し、1 週間ほどの期間を取ることをオススメします。

![Two-factor authenticationの強制化](/images/3tips-to-make-github-org-more-secure/github-require-two-factor-auth.png)
*Two-factor authenticationの強制化*

その後、設定を有効にするわけですが、1 週間以内にやってくれない人もいると思います。そういった人は、一旦上記設定を有効にし、組織から強制排除しても問題ありません。2FA の設定が完了したら連絡してもらうように共有し、もう一度招待をかける必要はありますが、所属 Team の設定は復元できるため、大した作業は不要です。

# Verified & approved domains を設定し、メール通知のドメインを制限する `(enterprise)`

個人のメールアドレスを使用しているユーザーに、組織の通知が送られてしまうと、組織の意図しない機密漏洩リスクが発生します。そこで、組織として Google Workspace 等で使用しているドメインを登録することで、そこで発行されたメールアドレスをもつユーザーにしか通知を受け取れないように設定できます。

設定は以下の公式ドキュメントを参考にしてください。

https://docs.github.com/en/organizations/keeping-your-organization-secure/restricting-email-notifications-for-your-organization

こちらに関しては、会社ドメインのメールアドレスを使用していないメンバーが組織から強制排除されることはありません。ただし、通知がこなくなるため、そちらの旨だけ伝え有効にすれば問題ありません。会社ドメインのメールアドレスを登録しているメンバーは、組織の People タブにて、そのメールアドレスが表示されます。登録していないメンバーの場合は、`No verified domain email` と表示されるため、ひと目で登録していないメンバーが分かるようになっています。

# SAML single sign-on を設定する `(enterprise)`

SAML を設定する利点はいくつかありますが、会社のレギュレーションに合わせた認証を組織に設定できることだと思っています。もちろん、他のサービス、例えば Slack Business+ 以上を契約していれば、そちらと認証基盤を統合できることも便利です。

メンバーが設定しなければならないパスワード長は必ず 20 文字以上で英数字記号を含む、であったり、2FA を必ず設定する必要がある、といったレギュレーションが会社にあると思います。そういった設定を、認証基盤を切り分けることでフレキシブルに設定できます。

こちらの記事では、Auth0 と GCP を使用して、Google Workspace に存在するユーザーのみを組織内にサインインできるように設定する方法を紹介します。

:::message
別のサービスを利用されたい場合は、以下の公式ドキュメントに設定方法が載っているので、そちらを参考にしてください。
https://docs.github.com/en/organizations/managing-saml-single-sign-on-for-your-organization/enabling-and-testing-saml-single-sign-on-for-your-organization
:::

## Auth0 をセットアップする

まず、Auth0 のアカウントを作成する必要があります。

https://auth0.com/jp

上記サイトで、メールアドレスを入力して、アカウントを作成してください。テナントの作成が必要な場合は、適宜作成してください。

テナントの画面が表示されたら、`Marketplace` へ移動し、`GitHub Enterprise Cloud` と検索します。

![Auth0 Marketplace](/images/3tips-to-make-github-org-more-secure/auth0-marketplace.png)

検索して出てきた Integration をクリックすると、詳細画面に移動するため、`Add Integration` ボタンをクリックしてください。

![Auth0 GitHub Enterprise Cloud](/images/3tips-to-make-github-org-more-secure/auth0-github-enterprise-cloud-page.png)

すると、以下の画像のページに移動するので、そのまま `Continue` をクリックします。

![Integrationのインストール画面](/images/3tips-to-make-github-org-more-secure/auth0-install-integration.png)

その後、Integration の設定画面に移動するので、下記の情報を登録してください。

* Callback URL: `https://github.com/orgs/${YOUR_GITHUB_ORG_NAME}/saml/consume`
* Audience: `https://github.com/orgs/${YOUR_GITHUB_ORG_NAME}`

![Integrationの設定画面](/images/3tips-to-make-github-org-more-secure/auth0-integration-settings.png)

最後に、下記画像のように SAML の設定パラメータが表示されるので、これらの情報を GitHub Organization の SAML 設定画面に設定する必要があります。

![Integrationのパラメータ](/images/3tips-to-make-github-org-more-secure/auth0-integration-configuration.png)

## Auth0 と GitHub を繋げる

GitHub の Organization ページから、`Organization security` に移動します。`SAML single sign-on` 節で、`Enable SAML authentication` をクリックします。すると、以下の画像のようになるため、上の画像部分で表示された SAML の設定パラメータを、GitHub の設定画面に入力します。GitHub の Public certificate 以外はそのままコピペで問題ありません。

| Auth0                         | GitHub             |
| ----------------------------- | ------------------ |
| Identity Provider Login URL   | Sign on URL        |
| Issuer                        | Issuer             |
| Identity Provider Certificate | Public certificate |

![GitHubのSAML設定画面](/images/3tips-to-make-github-org-more-secure/github-saml-sso.png)

Auth0 の Identity Provider Certificate は、URL として表示されているため、それを以下のコマンド等を使用して取得・コピーし、それを貼り付けてください。

```sh
$ curl https://dev-s0tkr2z1.us.auth0.com/pem | pbcopy
```

情報を貼り付けたら、`Test SAML configuration` ボタンを押して、正常に認証画面が表示されるか確認してください。正常に動作したのを確認できたら、`Save` ボタンを押して、設定を保存してください。

## Auth0 の設定を改善する

ここでは、Google Workspace に存在するユーザーのみを許可するように設定します。さらに、Auth0 側で特定のドメインのみを許容するように設定できるため、そちらも行います。また、セキュリティのために MFA を有効化します。こういった柔軟なカスタマイズを行える点が、SAML single sign-on の醍醐味かなと思います。

### 特定のドメインのみを許容する

Auth0 の管理画面から、`Auth Pipeline` -> `Rules` -> `Create` と選択し、出てきたテンプレートの中から、`Email domain whitelist` を選択します。スクリプト内に、`whitelist` 変数があるので、そちらの中身を Google Workspace で使用しているドメインに変更してください。

![Auth0 Email Domain Whitelist](/images/3tips-to-make-github-org-more-secure/auth0-email-domain-whitelist.png)

その後、`Save changes` をクリックし、Auth0 の Rules ページに戻ると、先程設定した `Email domain whitelist` が有効になっていることが確認できます。

![Auth0 Rules](/images/3tips-to-make-github-org-more-secure/auth0-rules.png)

### Google Workspace に存在するユーザーのみを許容する

#### OAuth 同意画面を作成する

社内用の GCP プロジェクトがあれば、そちらで問題ありません。Google Workspace の組織の下に配置されるようにプロジェクトを作成してください。

プロジェクトを作成したら、OAuth 同意画面を作成していきます。まず、`OAuth 同意画面` と検索し、そちらをクリックします。

![GCP OAuth 同意画面を検索](/images/3tips-to-make-github-org-more-secure/gcp-searching-oauth.png)

同意画面の設定に移動するため、そちらで `内部` を選択し、作成してください。Google Workspace 配下にプロジェクトが作成されていない場合、こちらを選択できないため、気をつけてください。

![GCP OAuth 同意画面を設定](/images/3tips-to-make-github-org-more-secure/gcp-configure-oauth.png)

次の画面では、適宜ご自身の情報を入力し、承認済みドメインには、`auth0.com` を入力してください。Auth0 のプランを有料プランにすれば、カスタムドメインを使用できるため、その場合はそちらのドメインを使用してください。

![GCP OAuth 同意画面の承認済みドメイン](/images/3tips-to-make-github-org-more-secure/gcp-oauth-approved-domain.png)

その後のスコープ等の設定は、そのままで進めていただいて問題ありません。

#### OAuth の認証情報を取得する

続いて、OAuth 認証情報を取得します。左側のタブから、`認証情報` -> `認証情報を作成` -> `OAuth クライアント ID` へ進んでください。

![GCP OAuth クライアント ID](/images/3tips-to-make-github-org-more-secure/gcp-oauth-client-id.png)

名前は適当なものを選択し、以下の情報を入力してください。

* 承認済みの JavaScript 生成元: `https://${YOUR_AUTH0_DOMAIN}`
* 承認済みのリダイレクト URI: `https://${YOUR_AUTH0_DOMAIN}/login/callback`

![GCP OAuth クライアント ID 設定](/images/3tips-to-make-github-org-more-secure/gcp-oauth-client-id-settings.png)

最後に、`作成` をクリックすると、クライアント ID とクライアントシークレットが表示されます。これらを、Auth0 側で設定する必要があるため、残しておいてください。

#### Auth0 に Google OAuth2 を設定する

Auth0 の管理画面から、`Authentication` -> `Social` -> `google-oauth2` へ進んでください。そちらで、`Client ID` 欄と `Client Secret` をそれぞれ先程表示されたものを入力し、それ以外はそのままで `Save changes` ボタンをクリックして変更を保存してください。

![Auth0 Google OAuth2 Setting](/images/3tips-to-make-github-org-more-secure/auth0-google-oauth2-settings.png)

Auth0 はデフォルトで Google OAuth2 認証が有効になっているため、このまま GitHub の SSO 画面にて、Google 認証が行えるようになっています。

### ユーザーネーム/パスワード認証を無効にする

Google Workspace 内に存在するユーザーのみを許可したい場合は、ユーザーネーム/パスワード認証を無効にしないと、存在しないユーザーでもアカウント作成・ログインができてしまいます。ユーザーネーム/パスワード認証を無効にしたくない場合は、アカウント作成のみを無効化できます。こちらの記事では、ユーザーネーム/パスワード認証自体を無効にします。

Auth0 の管理画面から、`Applications` -> `SSO Integrations` -> `GitHub Enterprise Cloud` -> `Connections` にアクセスしてください。そこに、`Username-Password-Authentication` とあるので、そちらのトグルをオフに変更すると、無効にできます。

![Auth0 ユーザーネーム/パスワード認証を無効化](/images/3tips-to-make-github-org-more-secure/auth0-integration-connections.png)

### MFA を有効にする

SSO は会社の情報を守るためのものなので、セキュリティはなるべく強固な方が良いでしょう。そのため、MFA を有効にすることをオススメします。Auth0 では、ユーザーに MFA を強制化でき、色々な認証方法を追加できます。こちらの記事では、One-time Password と WebAuthn を利用した生体認証を有効にします。

OTP と WebAuthn の両方を有効にした場合は、OTP の登録は必須、WebAuthn の登録は任意となり、その状態で WebAuthn を登録した場合は、そちらを優先的に使用します。そのため、OTP アプリを起動するのが面倒と思っている場合は、WebAuthn を登録することで、普段から OTP アプリを起動する必要がありません。

#### One-time Password を有効にする

Auth0 の管理画面から、`Security` -> `Multi-factor Auth` へ移動し、`One-time Password` のトグルをオンにします。

#### WebAuthn with FIDO Device Biometrics を有効にする

こちらは、俗に言う生体認証で、指紋認証や顔認証でのログインが可能になります。上記と同一のページで、`WebAuthn with FIDO Device Biometrics` に移動し、同じようにトグルをオンにします。

![Auth0 WebAuthn](/images/3tips-to-make-github-org-more-secure/auth0-webauthn.png)

---

上記の設定が完了すれば、以下の画像のように、OTP と WebAuthn が有効になっていると思います。

![Auth0 MFA](/images/3tips-to-make-github-org-more-secure/auth0-mfa.png)

#### MFA を強制化する

Auth0 では、MFA を強制化できます。選択的にしてしまうと設定しない人も現れるため、強制にする方が良いでしょう。

上記と同じページ内でそのまま下にスクロールすると、ポリシーを編集できるので、`Always` を選択し、`Save` をクリックして設定を保存します。

![Auth0 Require MFA](/images/3tips-to-make-github-org-more-secure/auth0-require-mfa.png)

## SAML single sign-on を強制化する

GitHub Organization のメンバーが SSO リンクにアクセスすると、SSO 認証情報と GitHub アカウントが紐づきます。SSO リンクは、SAML の設定画面に表示されているので、そちらを Slack 等で共有し、そちらのリンクからアクセスしてもらうように共有します。

基本的に１週間程の余裕をもって、SSO に移行してもらうようにすると良いでしょう。移行の進捗具合は、GitHub Organization ページの People 欄に、SSO フィルターが追加されているため、そちらで確認可能です。

![GitHub SSO Filter](/images/3tips-to-make-github-org-more-secure/github-sso-filter.png)

SAML single sign-on を強制化するにあたって、こちらも同様に 1 週間で移行してくれない人がいると思います。しかし、こちらも強制排除しても Team が復元できるため、問題ありません。また、SAML の場合は Outside Collaborator には影響はありません。SAML の設定画面から下にスクロールすると、強制化するためのボタンが表示されているため、こちらをクリックすると設定できます。

![GitHub Require SSO](/images/3tips-to-make-github-org-more-secure/github-require-saml-sso.png)

# GitHubアカウントに個人アカウントの使用を許可する

ここまでの設定を終えれば、個人アカウントの使用を許可しても良いでしょう。

まず、GitHub の利用規約として、無料アカウントを複数持つことは規約違反になります。そのため、個人アカウントを既に持っている人に対し、会社のアカウントを無料として発行することは利用規約違反になります。SSO 等を導入したことで、組織のセキュリティはある程度担保されているため、個人アカウントを使ってもよいというレギュレーションに更新できるかと思います。

エンジニアにとって、アカウント切り替えは非常に面倒なため、個人アカウントで会社にコミットできることは非常に嬉しいことです。ただし、個人アカウントを使用したくない人もいるため、こういった人には会社側で有料アカウントを発行し、そちらを組織に招待します。

今までであれば、会社で全てのメンバーのアカウントを発行・管理していたと思いますが、そういったコストを下げることができる点も利点かと思います。もちろん、全てのメンバーで会社アカウントとして統一している方が管理しやすい場合もあるので、メンバーにとっての利点も考慮した上で、レギュレーションを更新すると良いでしょう。
