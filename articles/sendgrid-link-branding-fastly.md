---
title: "Sendgrid のメール内リンクが SSL 証明書エラーになる問題"
emoji: "📨"
type: "idea"
topics: ["sendgrid", "fastly", "ssl"]
published: true
---

Sendgrid では、クリックトラッキングを有効にできます。
クリックトラッキングは、Sendgrid がメールを送信する前に、そのメール内に含まれる全てのリンクを Sendgrid 経由のリンクに置き換えることで Sendgrid 側でリンクのクリック率などを計測するための機能です。
その対象のリンクが、HSTS (HTTP Strict Transport Security) を有効にしている場合に、Sendgrid の用意するトラッキングリンクはデフォルトで HTTPS に対応していないために、SSL 証明書エラーとなります。

この問題に関しては、以下のブログでも言及されています。

https://soudai.hatenablog.com/entry/2019/07/16/082105

しかし、Fastly を使用している場合の設定方法に関しては示されていないため、こちらの記事で紹介させていただきます。

# Sendgrid の Link Branding に使用しているドメインを確認する

1. Sendgrid の [Sender Authentication ページ](https://app.sendgrid.com/settings/sender_auth) にアクセスする
2. `Link Branding` 欄に設定しているドメインをコピーしておく

`Link Branding` 用のドメインは、例えば以下のような感じに開発環境用と本番環境用が設定されているかと思います。
そういった、開発環境と本番環境それぞれに対して、Link Branding を設定している場合に対応する方法に関しても紹介します。

![Link Branding](/images/sendgrid-link-branding-fastly/link-branding.png)

# Sendgrid 用の Fastly サービスを作成する

https://docs.sendgrid.com/ui/sending-email/content-delivery-networks#using-fastly

https://support.sendgrid.com/hc/en-us/articles/1260802541709-SSL-Click-Tracking-Steps

一応、上記 2 つの記事に設定方法が示されていますが、少し情報不足のため、こちらの記事でも紹介いたします。

1. [Fastly Console](https://manage.fastly.com/services/all) に移動する
2. `Create a Delivery service` をクリックする
3. サービス名を例えば、`Sendgrid` に設定する
   ![Service Name](/images/sendgrid-link-branding-fastly/service-name.png)
4. `Domains` で、先程コピーしておいた `Link Branding` 用ドメインを貼り付ける
   ![Domains](/images/sendgrid-link-branding-fastly/domains.png)
5. `Origins` > `Hosts` へ移動し、`sendgrid.net` と入力し、`Add` をクリックする
   ![Hosts](/images/sendgrid-link-branding-fastly/hosts.png)
6. `Activate` をクリックして、サービスを有効化する

# Link Branding 用ドメインに SSL 証明書を発行する

こちらは、Sendgrid がやってくれるわけではなく、また、Fastly も勝手にやってくれるわけではありません。
Fastly を使用すれば、自動で証明書の更新をしてくれるため、自前で Let's Encrypt 等を作成・管理する必要がありません。

1. Fastly Console の [SSL 証明書発行のためのページ](https://manage.fastly.com/network/domains) へアクセスする (Secure > TLS management)
2. + マークのボタン (1 つでもドメインを登録している場合は、`Secure another domain` となっています) を押し、`Use certificates Fastly obtains for you` をクリックする
   ![Certificates](/images/sendgrid-link-branding-fastly/certificates.png)
3. Link Branding 用のドメインをそれぞれ入力し、`Submit` をクリックする
   ![Configure Certificates](/images/sendgrid-link-branding-fastly/configure-certificates.png)

上記の手順を行うと、以下のようにそれぞれ用の SSL Certificates が作成されていることが確認できます。

![Certificate for production](/images/sendgrid-link-branding-fastly/cert-for-prd.png)
![Certificate for development](/images/sendgrid-link-branding-fastly/cert-for-dev.png)

# SSL Click Tracking を有効にするために Support に連絡する

1. [Support ページ](https://support.sendgrid.com/hc/en-us/requests/new) にアクセスする
2. `Tell us about your issue` に、`Request to enable SSL Click Tracking` と入力する
3. 検索ボタンをクリックし、`Show more results` をクリック、`Continue` をクリックして、`Open a support request` をクリックする
4. 以下のようにリクエストを書き、`Submit` をクリックする
   ![Support](/images/sendgrid-link-branding-fastly/support.png)

しばらくすると、SSL Click Tracking を有効にしたよ、と連絡がくるので、それで完了になります。
もし上手く CDN を設定できていなかったら、その旨を教えてくれると思います。

# 最後に

以上で全てになります。
意外と大変なのと、Fastly に課金しなければいけない点が少し気になりますが、SSL 証明書のエラーが顧客側で表示されてしまうのは非常にマズイので、仕方無いところかと思います。

こちらの記事が参考になれば、幸いです。
