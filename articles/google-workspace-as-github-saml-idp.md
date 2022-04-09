---
title: "Google Workspace を GitHub の SAML IdP として使う"
emoji: "🦔"
type: "idea"
topics: ["googleworkspace", "github", "saml", "security"]
published: true
---

GitHub を会社として運用している場合、多くが Enterprise プランを契約されていると思います。
また、会社のセキュリティレギュレーションを満たすために SAML が広く導入されています。

以前私が書いた記事では、Auth0 を無料プランの範囲内で使用していました。

https://zenn.dev/matken/articles/3tips-to-make-github-org-more-secure#saml-single-sign-on-%E3%82%92%E8%A8%AD%E5%AE%9A%E3%81%99%E3%82%8B-(enterprise)

しかし、会社の規模が大きくなってくると、無料プランの範囲内では厳しくなってきます。
Pricing を見てみると、B2B プランとして契約が必要なため、最低でも $130/mo かかってしまいます。

![Auth0 Pricing - B2B](/images/google-workspace-as-github-saml-idp/auth0-pricing.png)
*Auth0 Pricing - B2B
Source: [Pricing - Auth0](https://auth0.com/pricing)*

前回は、Google Workspace が SAML IdP として使えることを知らず、無料で使用できる Auth0 を採用しました。
しかし、以下の記事を拝見している時、Google Workspace が使えることを知り、有料プランへの変更では無く既存に導入していた Google Workspace へ変更することにしました。

https://zenn.dev/ubie/articles/ee2ce9cc471f09#saml-sso%E3%81%A82fa%E3%81%AE%E5%BC%B7%E5%88%B6

# SAML IdP 変更の流れ

GitHub 上から、一旦 SAML をオフにする必要があります。
いきなり `Public certificate` 等を変更して保存してしまうと、GitHub 側に保存されている、SSO identity が、Auth0 のものと Google Workspace のもので合わないことになります。
こうなると、最悪の場合、全員が組織から弾かれてしまい、誰も入れなくなってしまう可能性があるためです。
SAML 認証を必須化していれば、もう誰もログインできないため、サポートデスクのお世話になるかもしれません。

:::message
GitHub 側で移行の手順が明記されていないため、このような対応をしましたが、設定変更時に SSO identity を一度削除する、といった処理がなされている可能性もあります。
その場合は、シンプルに設定を変更し、`Save` すれば問題無いかと思います。
:::

以下の手順で、一旦 SAML をオフにすることで、GitHub 上から SSO identity を削除でき、スムーズに移行できます。

1. GitHub 上の、SAML single sign-on の設定ページから、`Enable SAML authentication` のチェックマークを一旦外し、設定を `Save` する
2. Google Workspace で GitHub 用の SAML アプリを設定する
3. Google Workspace 側で表示された `Public certificate` 等を GitHub に登録する
4. 一定期間の移行期間を再度用意し、SAML SSO を強制化する

# Google Workspace を GitHub の SAML IdP として設定する

これ以降の作業をするには、Google Workspace の Admin 権限が必要です。
一応設定方法は以下のリンクに載っているのですが、少しややこしいので、こちらでも説明いたします。

https://support.google.com/a/answer/7553415?hl=ja

1. 「[ウェブアプリとモバイルアプリページ](https://admin.google.com/ac/apps/unified)」へ移動する
2. 「アプリを追加」>「アプリを検索」をクリックする
   ![Step 1](/images/google-workspace-as-github-saml-idp/step1.png)
3. 「GitHub Business」を検索し、選択する
   ![Step 2](/images/google-workspace-as-github-saml-idp/step2.png)
4. 表示された情報をそれぞれコピーする
5. 「続行」をクリックする
6. `{your-organization}` を GitHub の Organization ID に置き換える
7. 「続行」をクリックする
8. Google Workspace と GitHub を繋げる

GitHub の Organization ページから、`Organization security` に移動します。`SAML single sign-on` 節で、`Enable SAML authentication` をクリックします。
すると、以下の画像のようになるため、上の画像部分で表示された SAML の設定パラメータを、GitHub の設定画面に入力します。
「SHA-256 フィンガープリント」は使用しません。

| Google Workspace              | GitHub             |
| ----------------------------- | ------------------ |
| SSO の URL                    | Sign on URL        |
| エンティティ ID                 | Issuer             |
| 証明書                         | Public certificate |

![GitHubのSAML設定画面](/images/3tips-to-make-github-org-more-secure/github-saml-sso.png)

情報を貼り付けたら、`Test SAML configuration` ボタンを押して、正常に認証画面が表示されるか確認してください。
正常に動作したのが確認できたら、`Save` ボタンを押して、設定を保存してください。

# 最後に

以上で設定は完了です。
残りのタスクとしては、「SAML single sign-on を強制化する」ことや、「GitHub アカウントに個人アカウントの使用を許可する」などが挙げられます。

* SAML single sign-on を強制化する

https://zenn.dev/matken/articles/3tips-to-make-github-org-more-secure#saml-single-sign-on-%E3%82%92%E5%BC%B7%E5%88%B6%E5%8C%96%E3%81%99%E3%82%8B

* GitHub アカウントに個人アカウントの使用を許可する

https://zenn.dev/matken/articles/3tips-to-make-github-org-more-secure#github%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E3%81%AB%E5%80%8B%E4%BA%BA%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E3%81%AE%E4%BD%BF%E7%94%A8%E3%82%92%E8%A8%B1%E5%8F%AF%E3%81%99%E3%82%8B

Auth0 よりも簡単にセットアップできたかなと思います。
会社のセキュリティレギュレーションに基づいた認証ルール（MFA の強制等）を、Google Workspace で一元管理できるのが非常に良い点だと感じました。
これほど簡単なのであれば、GitHub 側でも公式にサポートを明言してほしいですね。
