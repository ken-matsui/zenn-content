---
title: "認証が必要な Cloud Run を別サービスから叩く方法"
emoji: "🔑"
type: "tech"
topics: ["cloudrun", "authentication"]
published: true
---

セキュリティ上の観点から、Cloud Run を内部からのみしか叩けないサービスにしたい時、Ingress 設定を「内部のみ」にするか、認証設定を「認証が必要」にするかのどちらか or どちらもになると思います。
Ingress の説明には以下のように記載されており、例えば Cloud Functions から Cloud Run を叩くといった構成の場合だと、「内部のみ」に設定して叩けそうですが、何故かエラーで叩けませんでした。

> Cloud Run サービスへのネットワーク アクセスを制限します。 同じプロジェクト内または VPC SC 境界内にある VPC ネットワーク、Pub/Sub、Eventarc からのリクエストは、内部リクエストとみなされます。 IAM 起動元の権限は引き続き適用されます。

もしかすると、実行元が Cloud Functions の場合は、[サーバーレス VPC アクセス](https://cloud.google.com/vpc/docs/configure-serverless-vpc-access) を構成する必要があるのかもしれません。
個人的なプロジェクトの場合、なるべく簡素な構成にしたいため、こういった VPC Connector の採用は見送りました。

そのため、認証設定を「認証が必要」にすることで、Cloud Run が勝手に叩かれないようにしました。

![Settings](/images/invoke-auth-required-cloud-run/settings.png)

# 認証が必要な Cloud Run を叩く

以下の流れで、認証が必要な Cloud Run を叩けるように設定していきます。

## サービスアカウントにロールを付与する

まずは、対象の Cloud Run を叩く元が使用するサービスアカウントに、`Cloud Run 起動元` というロールを付与することで、叩くための権限を設定します。

1. [Cloud Run ページ](https://console.cloud.google.com/run) へ移動する
2. 「認証が必要」と設定した Cloud Run サービスの左側に表示されている、チェックボックスにチェックを入れる
3. 右側に「プリンシパルを追加」ボタンが表示されているので、それをクリックする
4. Cloud Run を叩く元が使用するサービスアカウントを入力する（Cloud Functions であれば、`${YOUR_PROJECT_ID}@appspot.gserviceaccount.com`）
5. 「ロールを選択」ドロップダウンで、`Cloud Run 起動元` (`roles/run.invoker`) ロールを選択する
6. 「保存」をクリックする

## リクエストヘッダーを作成する

Cloud Functions から認証ヘッダーを Cloud Run に渡して実行できるようにします。
例えば Python であれば、以下のようなコードになります。

```python
audience = "https://my-cloud-run-service.run.app/"

auth_req = google.auth.transport.requests.Request()
id_token = google.oauth2.id_token.fetch_id_token(auth_req, audience)
headers = {"Authorization": f"Bearer {id_token}"}
response = requests.post(
    audience + "world", headers=headers, json={"message": "Hello"}
)
```

GCP のサービスであれば、サービスアカウントが自動で設定されているため、それを使用して認証ヘッダーを作成しています。

# 最後に

こちらの記事は、以下の記事を参考にしました。

https://cloud.google.com/run/docs/authenticating/service-to-service

Python の場合は、`urllib` が使われているので、そちらを `requests` を使うコードに変更しました。
また、リクエスト先の URL は `audience` と同じで良いのかよく分からなかったのですが、結果として、同じで大丈夫でした。

細かい部分や、他のケース・プログラミング言語に関しては、上記の記事が参考になるかと思います。
こちらの記事も合わせて参考になれば幸いです。
