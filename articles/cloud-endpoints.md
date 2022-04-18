---
title: "Cloud Endpoints を活用する"
emoji: "🌈"
type: "tech"
topics: ["gcp", "cloudendpoints", "envoy"]
published: true
---

# Cloud Endpoints とは

GCP には、Cloud Endpoints というサービスがあります。

https://cloud.google.com/endpoints

こちらは以下のような使い方ができます。

* Cloud Endpoints を認証を担う REST プロキシ
* その先に、バックエンドサービス

Cloud Endpoints は、Extensible Service Proxy V2（ESPv2）をデプロイすることで動作します。
今どきのサービスだと、[Apigee](https://cloud.google.com/apigee) や [Envoy](https://www.envoyproxy.io/) を使用されていることが多いと思いますが、その ESPv2 は Envoy ベースで実装されています。

https://cloud.google.com/endpoints/docs/openapi/migrate-to-esp-v2#unsupported_use_cases

# Cloud Endpoints の利点・欠点

私的に Cloud Endpoints を実際に、会社のプロダクト用として運用してみた時に感じた利点は以下の通りです。

1. OpenAPI ファイルと実際のサービスプロキシを同期させることができる
2. 認証をバックエンド実装に任せる必要がない

また、以下の欠点を感じました。

1. OpenAPI 2.0 にしか対応していない
2. 認証情報をバックエンド側で扱う時にややこしい
3. Cloud Endpoints Portal が提供終了

欠点の方が多いように見えますが、人によれば些細なことですので、利点の方が上回ることの方が多いかと思います。
また、1 番目の欠点以外は解決できます。

# 欠点に関して

## 1. OpenAPI 2.0 にしか対応していない

IssueTracker を見る限り、対応されそうにありません。

https://issuetracker.google.com/issues/78271318?pli=1

OpenAPI 3.0 で書き、それから 2.0 に変換する方法も考えてみましたが、結果的には難しく、諦めました。
そのため、Cloud Endpoints を使用するには、強制的に OpenAPI 2.0 を使用させられます。
こちらが難しい場合は、Cloud Endpoints を使用すること自体諦める方が良いかもしれません。

## 2. 認証情報をバックエンド側で扱う時にややこしい

Cloud Endpoints は認証を処理する際に、`Authorization` ヘッダーを以下の 3 つのヘッダーに変換します。

1. `Authorization`: 以下のような GCP サービスとしての JWT トークンに置換される
   ```json
   {
      "aud": "https://api-backend.example.dev",
      "azp": "${service_user_id}",
      "email": "${service_id}-compute@developer.gserviceaccount.com",
      "email_verified": true,
      "exp": 1622137489,
      "iat": 1622133889,
      "iss": "https://accounts.google.com",
      "sub": "${service_user_id}"
    }
   ```
2. `X-Forwarded-Authorization`: Cloud Endpoints に到達した `Authorization` ヘッダーの内容がそのまま入る
3. `X-Endpoint-API-UserInfo`: Cloud Endpoints に到達した `Authorization` ヘッダーの内容をそのまま Base64 で encode した内容が入る

> 参考

https://github.com/GoogleCloudPlatform/esp-v2/pull/141

この点が非常にややこしく、デプロイするまで分からないため、何度もデプロイを繰り返し、どういったヘッダーが生成されているのかを調べました。

Auth0 等のユーザー情報に加えて、バックエンド側でもユーザー情報を持っていると思います。
それを紐付けるために、JWT トークンの `sub` を識別子として一意に紐付けることが多いのかなと思います。

そのときは以下のように、`X-Forwarded-Authorization` ヘッダーをパースします。

```python
import base64
import json
from typing import Any, Dict, Tuple


def load_json_from_jwt_token_fragment(jwt_token: str) -> Any:
    return json.loads(
        # ref: https://stackoverflow.com/a/49459036
        base64.urlsafe_b64decode(jwt_token + "=" * (-len(jwt_token) % 4))
    )

def extract_jwt_header_and_payload(
    jwt_token: str,
) -> Tuple[Dict[str, Any], Dict[str, Any]]:
    jwt_header, jwt_payload = map(
        load_json_from_jwt_token_fragment,
        jwt_token.split(".")[:-1],  # drop the last element (verify signature)
    )
    return jwt_header, jwt_payload
```

そして、その返り値である、`jwt_payload` を使用して `sub` を取り出し、ユーザー情報を取得するという流れになります。

```python
import binascii


try:
    jwt_header, jwt_payload = extract_jwt_header_and_payload(jwt_token)
    sub = jwt_payload["sub"]
except binascii.Error:
    print("Provided jwt token could not be parsed; it might be invalid form.")
```

:::details Django と Rest Framework では Authentication Class が作成できるので、以下のように実装できます。

```python: myapi/authentication.py
import binascii
from typing import Any, Dict, Final, Optional, Tuple

from django.core.exceptions import ObjectDoesNotExist
from django.http.request import HttpHeaders
from rest_framework.authentication import BaseAuthentication
from rest_framework.request import Request
from result import Err, Ok, Result
from myapi.models import User


def parse_jwt_token(
    headers: HttpHeaders,
) -> Result[Tuple[str, Dict[str, Any], Dict[str, Any]], None]:
    if Authentication.before_authenticate(headers).is_err():
        return Err(None)

    auth_header_value: str = headers[Authentication.AUTH_HEADER]
    if not auth_header_value.startswith("Bearer "):
        print("Provided jwt token does not start with Bearer.")
        return Err(None)
    jwt_token: str = auth_header_value.replace("Bearer ", "")

    try:
        jwt_header, jwt_payload = extract_jwt_header_and_payload(jwt_token)
    except binascii.Error:
        print("Provided jwt token could not be parsed; it might be invalid form.")
        return Err(None)
    return Ok((jwt_token, jwt_header, jwt_payload))

class Authentication(BaseAuthentication):
    AUTH_HEADER: Final[str] = "X-Forwarded-Authorization"
    USERINFO_HEADER: Final[str] = "X-Endpoint-API-UserInfo"

    @staticmethod
    def before_authenticate(headers: HttpHeaders) -> Result[None, None]:
        auth_header: Final[str] = Authentication.AUTH_HEADER
        userinfo_header: Final[str] = Authentication.USERINFO_HEADER
        warning_str: str = "{} not found in request.headers"

        if auth_header not in headers:
            print(warning_str.format(auth_header))
            return Err(None)
        elif userinfo_header not in headers:
            # This header is just used for validating its request is from Cloud Endpoints.
            print(warning_str.format(userinfo_header))
            return Err(None)
        else:
            return Ok(None)

    def authenticate(self, request: Request) -> Optional[Tuple[User, Dict[str, Any]]]:
        jwt_result = parse_jwt_token(request.headers)
        if isinstance(jwt_result, Err):
            return None
        jwt_token, jwt_header, jwt_payload = jwt_result.unwrap()

        try:
            user: User = User.get(sub=jwt_payload["sub"])
        except ObjectDoesNotExist:
            # When the requested user that made with Auth0 does not exist in Django's DB
            return None

        return user, {
            "jwt_token": jwt_token,
            "jwt_header": jwt_header,
            "jwt_payload": jwt_payload,
        }

    def authenticate_header(self, request: Request) -> str:
        # To return `401: Unauthorized` instead of `403: Permission Denied`
        return "Access to the authentication required site"
```

```python: settings.py
REST_FRAMEWORK: Dict[str, Any] = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "myapi.authentication.Authentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

すると、以下のように `request.user` としてユーザー情報を使用できるようになります。

```python: mysite/views.py
class UserRetrieveAPIView(RetrieveAPIView):
    def get(self, request: Request, *args: Any, **kwargs: Any) -> Response:
        serializer = self.get_serializer_class()(request.user)
        return Response({"user": serializer.data}, status=HTTP_200_OK)
```
:::

## 3. Cloud Endpoints Portal が提供終了

Cloud Endpoints Portal とは、デプロイ時に合わせて [Swagger UI](https://swagger.io/tools/swagger-ui/) や [Redoc](https://github.com/Redocly/redoc) のようなページを自動で生成してくれるものです。
Cloud Endpoints Portal の利点は、IAM を使ってそのポータルへのアクセスを管理できる点です。

![Endpoints Portal](/images/cloud-endpoints/endpoints-portal.png)
*Source: [Cloud Endpoints Portal overview](https://cloud.google.com/endpoints/docs/frameworks/dev-portal-overview)*

しかし、Swagger UI や Redoc と比較すると圧倒的に見づらいです。

そういったこともあり、非推奨になるみたいで、2023 年 3 月 21 日をもってデプロイされなくなります。

https://cloud.google.com/endpoints/docs/deprecations/endpoints-portal-deprecation?hl=ja

ただ、API ドキュメントを開発チームに展開する時は、Redoc 等でどこかにデプロイされていると見やすいです。
この記事では、[Private GitHub Pages](https://github.blog/jp/2021-01-25-access-control-for-github-page/) を使用して、Redoc としてデプロイすることをおすすめします。

```yaml: .github/workflows/api-docs.yml
name: 'API Docs'

on:
  push:
    branches: [ main ]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3.0.0

      - name: Build for Redoc
        run: npx redoc-cli bundle ./openapi.yaml

      - name: Move the Redoc output file to dist/
        run: mv redoc-static.html dist/index.html

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./dist
```

上記の Action を実行した後は、レポジトリページ > Settings > Pages > Source で `gh-pages` ブランチを選択すれば、GitHub Pages に表示されます。

API ドキュメントを閲覧するのは、エンジニアしかいないはずで、その場合は、GitHub に対してアクセス権があるはずです。
そのアクセス権を使用して、Private GitHub Pages 側で認証し、適切なユーザーのみが API ドキュメントを閲覧できます。

# Cloud Endpoints 用の `openapi.yaml` を作成する

Cloud Endpoints をデプロイしていくために、まずは、`openapi.yaml` を作成しましょう。

1. まず、3.0 を使用できないため、バージョンを 2.0 に設定します
   ```yaml: openapi.yaml
   swagger: "2.0"
   ```
2. 次に、`info` 欄には、`version` を[指定できます](https://cloud.google.com/endpoints/docs/openapi/versioning-an-api)。ここでは、`1.0.0-beta` とします
   ```yaml: openapi.yaml
   info:
     version: 1.0.0-beta
   ```
3. `host` として、Cloud Endpoints が API として振る舞うための URL を指定します
   ```yaml: openapi.yaml
   host: "api.example.dev"
   ```
4. `basePath` にバージョンを指定すると、API のバージョニングが簡単に行えるため、設定することをおすすめします
   そのままシンプルにスラッシュのみでも問題ありません。
   ```yaml: openapi.yaml
   basePath: /v1beta
   ```
5. CORS を設定します
   ```yaml: openapi.yaml
   # Ref: https://cloud.google.com/endpoints/docs/openapi/support-cors
   x-google-endpoints:
     - name: "api.wiz-dom.dev"
       allowCors: True
   ```
6. バックエンドと繋ぎ込みを行います
   例えば、Django 等で書かれたサーバーになります。
   `path_translation` に `APPEND_PATH_TO_ADDRESS` を指定することで、そのままパスをバックエンドに渡してくれます。
   基本的には、こちらを使用することが多いと思います。
   `path_translation` の詳細は[こちら](https://cloud.google.com/endpoints/docs/openapi/openapi-extensions#understanding_path_translation)に書かれているので参考にしてください。
   ```yaml: openapi.yaml
   # Ref: https://cloud.google.com/endpoints/docs/openapi/openapi-extensions
   x-google-backend:
     address: "https://api-backend.example.dev"
     path_translation: APPEND_PATH_TO_ADDRESS
   ```
7. 認証を使用する
   Auth0 を使用する場合は、以下のように記述できます。
   ```yaml: openapi.yaml
   securityDefinitions:
     auth0_jwk:
       authorizationUrl: "https://auth.example.dev/authorize"
       flow: "implicit"
       type: "oauth2"
       x-google-issuer: "https://auth.example.dev/"
       x-google-jwks_uri: "https://auth.example.dev/.well-known/jwks.json"
       x-google-audiences: "https://api.example.dev"

   security:
     - auth0_jwk: []
   ```
8. （Optional）リクエスト制限をかける
   例えば、Read 系のリクエストを 1 分につき 1000 回に制限し、Write 系のリクエストを 1 分につき 50 回に制限したい時は、以下のように記述できます。
   ```yaml: openapi.yaml
   # Ref: https://cloud.google.com/endpoints/docs/openapi/quotas-configure
   x-google-management:
     metrics:
       - name: "read-requests"
         displayName: "Read requests"
         valueType: INT64
         metricKind: DELTA
       - name: "write-requests"
         displayName: "Write requests"
         valueType: INT64
         metricKind: DELTA
     quota:
       limits:
         - name: "read-limit"
           metric: "read-requests"
           unit: "1/min/{project}"
           values:
             STANDARD: 1000
         - name: "write-limit"
           metric: "write-requests"
           unit: "1/min/{project}"
           values:
             STANDARD: 50
   ```

これでベースラインは完成なので、後は、パスを追加していきます。
例えば以下のようになると思います。

```yaml: openapi.yaml
paths:
  /users:
    get:
      operationId: users.get
      tags:
        - user
      x-google-quota:
        metricCosts:
          "read-requests": 1
```

`operationId` は、Cloud Endpoints にとって必須なため、付けましょう。
`<path>.<method>` の形式がシンプルで、他と被ることが無いためおすすめです。

先程の 8 のステップでリクエスト制限を付けた場合は、`x-google-quota` を使用して API 毎に設定します。
この API は、`read-requests` を 1 消費するため、1 分間に 1000 回以上呼び出すことができません。

例えば、user 一覧を取得する API が user 一人の情報を取得する API よりもコストがかかると考えた場合は、`"read-requests": 2` とすることで、1 分間に 500 回以上呼び出せないように制限できます。
詳しい設定に関しては、以下を参考にしてください。

https://cloud.google.com/endpoints/docs/openapi/quotas-configure

# Cloud Endpoints にデプロイする

https://cloud.google.com/endpoints/docs/openapi/get-started-cloud-run

上記の記事に従って、各種セットアップを行ってください。

# GitHub Actions をセットアップする

Cloud Endpoints にデプロイできたと思いますが、継続的にデプロイしたいと思うので、それ用の GitHub Actions を用意します。
また、OpenAPI の書き方エラー等をチェックするための GitHub Actions もついでに用意します。

## 1. Cloud Endpoints へ自動でデプロイする

```yaml: .github/workflows/cloud-endpoints.yml
name: 'Cloud Endpoints'

on:
  push:
    branches: [ main ]

env:
  ESP_FULL_VERSION: 2.25.0

# Ref: https://cloud.google.com/endpoints/docs/openapi/get-started-cloud-run
jobs:
  deploy:
    permissions:
      contents: 'read'
      id-token: 'write'

    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3.0.0

      - id: auth
        uses: google-github-actions/auth@v0
        with:
          workload_identity_provider: 'projects/${{ secrets.GCP_PROJECT_NUMBER }}/locations/global/workloadIdentityPools/github-actions/providers/gha-provider'
          service_account: 'github-actions-espv2-cloudrun@${{ secrets.GCP_PROJECT_ID }}.iam.gserviceaccount.com'

      - name: Deploy the Endpoints configuration
        run: gcloud endpoints services deploy ./openapi.yaml

      - name: Get the latest config id
        run: |-
          echo "CONFIG_ID=$(gcloud endpoints configs list \
            --service=${{ secrets.ENDPOINTS_SERVICE_HOST }} \
            --sort-by='~CONFIG_ID' \
            --limit=1 \
            | tail -1 | cut -d' ' -f1)" >> "$GITHUB_ENV"
      - name: Download gcloud_build_image script
        run: |
          wget https://raw.githubusercontent.com/GoogleCloudPlatform/esp-v2/master/docker/serverless/gcloud_build_image
          chmod +x ./gcloud_build_image
      - name: Build a new ESPv2 image
        run: ./gcloud_build_image -s ${{ secrets.ENDPOINTS_SERVICE_HOST }} -c "$CONFIG_ID" -p ${{ secrets.GCP_PROJECT_ID }}

      # https://cloud.google.com/endpoints/docs/grpc/specify-esp-v2-startup-options?hl=JA#cors
      - name: Deploy the ESPv2 container to Cloud Run
        run: |-
          gcloud run deploy wizdom-endpoints \
            --image=gcr.io/${{ secrets.GCP_PROJECT_ID }}/endpoints-runtime-serverless:"$ESP_FULL_VERSION"-${{ secrets.ENDPOINTS_SERVICE_HOST }}-"$CONFIG_ID" \
            --set-env-vars=ESPv2_ARGS=^++^--cors_preset=basic++--cors_expose_headers=Content-Disposition \
            --allow-unauthenticated \
            --platform=managed --project=${{ secrets.GCP_PROJECT_ID }} \
            --region=${{ secrets.GCP_REGION }}
```

以下の Secrets を設定する必要があります。

* GCP_REGION: e.g. `asia-northeast1`
* GCP_PROJECT_ID: e.g. `hoge-fuga-12345`
* GCP_PROJECT_NUMBER: e.g. `123456789101`
* ENDPOINTS_SERVICE_HOST: e.g. `gateway-12345-uc.a.run.app` (デプロイ先の Cloud Run Service Host)

## 2. Cloud Endpoints へデプロイ可能か検証する

サービスの設定を検証するのみで、実際にはデプロイしません。
PR 上で走らせ、`main` ブランチへの Branch Protection Rule の requirements に追加するのがおすすめです。

```yaml: .github/workflows/cloud-endpoints-validator.yml
name: 'Cloud Endpoints'

on: pull_request

# Ref: https://cloud.google.com/endpoints/docs/openapi/get-started-cloud-run
jobs:
  validate:
    permissions:
      contents: 'read'
      id-token: 'write'

    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3.0.0

      - id: auth
        uses: google-github-actions/auth@v0
        with:
          workload_identity_provider: 'projects/${{ secrets.GCP_PROJECT_ID_NUMBER }}/locations/global/workloadIdentityPools/github-actions/providers/gha-provider'
          service_account: 'github-actions-espv2-cloudrun@${{ secrets.GCP_PROJECT_ID }}.iam.gserviceaccount.com'

      - name: Deploy the Endpoints configuration
        run: gcloud endpoints services deploy ./openapi.yaml --validate-only
```

## 3. OpenAPI ファイルを検証する

OpenAPI ファイル自体も一応検証しておくと安心です。

```yaml: .github/workflows/swagger.yml
name: 'Swagger'

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3.0.0

      - name: Validate openapi
        run: npx swagger-cli validate ./openapi.yaml
```

## 4. [Textlint](https://github.com/textlint/textlint) でドキュメントの文章を検証する

Textlint で日本語文章を検証しておくと、尚安心です。

```yaml: .github/workflows/textlint.yml
name: 'Textlint'

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  textlint:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3.0.0

      - name: Install dependencies
        run: npm i

      - name: Convert OpenAPI to Markdown
        run: npx openapi-markdown -i ./openapi.yaml

      - name: Run textlint
        run: npm run lint
```

```json: package.json
{
    "scripts": {
      "lint": "textlint './openapi.md'"
    },
    "devDependencies": {
      "textlint": "12.1.1",
      "textlint-filter-rule-comments": "1.2.2",
      "textlint-rule-preset-ja-spacing": "2.2.0",
      "textlint-rule-preset-ja-technical-writing": "7.0.0"
    }
}
```

```yaml: .textlintrc.yml
plugins:
  '@textlint/markdown':
    extensions: [".md"]

rules:
  preset-ja-technical-writing:
    no-exclamation-question-mark:
      allowFullWidthExclamation: true
      allowFullWidthQuestion: true
    no-doubled-joshi:
      strict: false
      allow:
        - "か"  # 助詞のうち「か」は複数回の出現を許す (e.g.: するかどうか)
        - "に"  # e.g.: 内容を元に `user` テーブルに
    no-doubled-conjunction: false
    ja-no-mixed-period: false  # 文末の"。"強制を無効化
    sentence-length: false  # 100文字数制限の無効化
  preset-ja-spacing:
    ja-space-between-half-and-full-width:
      space: "always"
      exceptPunctuation: true
    ja-space-around-code:
      before: true
      after: true

filters:
  comments: true
```

# 最後に

以上で全て完了となります。

最初は色々とセットアップが大変ですが、一度セットアップすれば、現在まで一度も問題が起きていません。
色々と便利なので、是非試してみてください。
この記事が参考になれば幸いです。
