---
title: "Slack を活用して GitHub でのタスク対応漏れを防ぐ GitHub Actions"
emoji: "🤖"
type: "idea"
topics: ["github", "githubactions", "slack"]
published: true
---

GitHub 公式の Slack アプリだと、情報が多かったり少なかったり、他のユーザーの通知で自分に関係するタスクが流れてしまいがちです。
そういった点で複数人開発には、公式アプリは向いていないなと思っていました。

他の Slack アプリも探してみたものの、あまり良いものが見つかりませんでした。

そこで、個人的に必要だと思った通知内容だけ切り出して、それらを通知するための GitHub Action を開発してみました。
以下のリンクが、今回開発したソースコードのレポジトリになります。

https://github.com/ken-matsui/notify-slack

# 通知内容

以下が全ての通知タイプで、これ以外の種類は届かないようになっています。

## 全てのユーザー

* PR もしくは、Issue でメンションを受け取った時
  ![Mentioned](/images/notify-slack-action/mentioned.png)

## レビュアー

* レビューリクエストを受け取った時
  ![Received Review](/images/notify-slack-action/received-review.png)

## デベロッパー

* レビューをリクエストした時
  ![Requested Review](/images/notify-slack-action/requested-review.png)
* レビュー結果が返ってきた時
  * Approved
    ![Approved](/images/notify-slack-action/approved.png)
  * Requested changes
    ![Requested Changes](/images/notify-slack-action/requested-changes.png)
  * Commented
    ![Commented](/images/notify-slack-action/commented.png)
* 作成した PR がマージされた時
  ![Merged](/images/notify-slack-action/merged.png)

# 設定方法

まず、導入したい Workspace に Slack App を作成し、それの OAuth Access Token を action に渡してあげることで、実行できるようにします。

## Slack App を作成する

以下の手順で作成します。

1. Slack の [Your Apps ページ](https://api.slack.com/apps) に移動する
2. `Create New App` をクリックし、`From scratch` を選択する
   ![Step 2](/images/notify-slack-action/step2.png)
3. `GitHub Notification` 等を `App Name` に入力し、Workspace を選択後に `Create App` をクリックする
   ![Step 3](/images/notify-slack-action/step3.png)
4. `Add features and functionality` から、`Permissions` をクリックする
   ![Step 4](/images/notify-slack-action/step4.png)
5. `Scopes` の `Bot Token Scopes` から、`chat:write` を選択する
   ![Step 5](/images/notify-slack-action/step5.png)
6. `Basic Information` へ戻り、`Install to Workspace` をクリックする
   ![Step 6](/images/notify-slack-action/step6.png)
7. `Allow` をクリックする
   ![Step 7](/images/notify-slack-action/step7.png)
8. `OAuth & Permissions` ページで、 `Bot User OAuth Token` をコピーする
   ![Step 8](/images/notify-slack-action/step8.png)

## GitHub レポジトリに action ファイルと 設定ファイルを用意する

以下の 2 点を用意してください。
action ファイルはそのままコピーで問題ありません。
userlist ファイルは、環境に合わせて編集してください。

1. ```yaml: .github/workflows/notify-slack.yml
   name: GitHub Notification

   on:
     pull_request:
       types: [review_requested, closed]
     pull_request_review:
       types: [submitted]
     issue_comment:
     pull_request_review_comment:

   jobs:
     notify:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v2

         - uses: ken-matsui/notify-slack@v1.0.0
           with:
             slack_oauth_access_token: ${{ secrets.SLACK_OAUTH_ACCESS_TOKEN }}
    ```
2. ```toml: .github/userlist.toml
   [[users]]
   github = "ken-matsui"
   slack = "UXXXXXXXXXX"

   [[users]]
   github = "ken-matsui-2"
   slack = "UXXXXXXXXXX"
   ```

`github` は、GitHub の User ID を入力し、`slack` は、Slack の User ID を入力します。
それを 1 つのユーザーとして解釈します。

Slack の User ID の取得方法は以下のリンクが参考になります。

https://www.workast.com/help/articles/61000165203/

後は、正しく設定できていれば、GitHub 上での作業に対して、それぞれ最初に示した画像のような通知が届くようになっているかと思います。

# 最後に

どうしても自分のタスクと関係の無い通知が同じチャンネルに来ていると、タスクを見逃しやすくなってしまいます。
そこをこちらの action では、DM として切り分け、通知内容も絞っているため、タスクが流れてしまうことを防げます。
完了したタスクには✅絵文字を付けるなど、お好みの運用をしていただければと思います。

基本的には、会社の Slack Workspace に導入し、それぞれの会社レポジトリに action ファイルを置いていくといった運用になるかなと思います。
もちろん個人として使っていただくことも可能なので、活用していただけると幸いです。
