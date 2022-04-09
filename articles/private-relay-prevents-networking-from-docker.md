---
title: "iCloud Private Relay を有効にすると Docker から HTTP リクエストができない問題"
emoji: "🚫"
type: "tech"
topics: ["macos", "docker", "icloud"]
published: true
---

ある時から、macOS の docker 上で apt コマンドが以下のようなエラーで失敗するようになりました。

```
W: Failed to fetch http://ports.ubuntu.com/ubuntu-ports/dists/focal/InRelease  Could not connect to ports.ubuntu.com:80 (...), connection timed out Could not connect to ports.ubuntu.com:80 (...), connection timed out
W: Failed to fetch http://ports.ubuntu.com/ubuntu-ports/dists/focal-updates/InRelease  Unable to connect to ports.ubuntu.com:http:
W: Failed to fetch http://ports.ubuntu.com/ubuntu-ports/dists/focal-backports/InRelease  Unable to connect to ports.ubuntu.com:http:
W: Failed to fetch http://ports.ubuntu.com/ubuntu-ports/dists/focal-security/InRelease  Unable to connect to ports.ubuntu.com:http:
W: Some index files failed to download. They have been ignored, or old ones used instead.
```

結論としては、iCloud Private Relay を macOS 上で有効にしていたことが原因でした。
以下のイシューの通り、iCloud Private Relay をオフにすることで解決できます。

https://github.com/docker/for-mac/issues/6030#issuecomment-992590521

ここで気をつけないといけないのが、設定 > Apple ID > Private Relay のチェックマークを外すだけでは、オフにはならないということです。
これに気づかず、チェックマークを外したにも関わらず上記問題が直らなかったため、別のことが原因だと思い、半日近く関係の無い解決策を試していました。

![demo](/images/private-relay-prevents-networking-from-docker/demo.gif)

キチンとオフにするには、Private Relay 欄の、`Options...` ボタンを押し、右上の `Turn Off...` ボタンをクリックして初めてオフになります。

逆に、オンにしたい時は、チェックマークを入れるだけで OK です。
チェックマークを外す時に、以下の画像のようなダイアログが表示されることもあり、それに従う場合は上手くオフになります。
ダイアログが出ないときは、オフにされていないので、気をつけましょう。

![dialog](/images/private-relay-prevents-networking-from-docker/dialog.png)
*たまに出てくるダイアログ*

:::message
今後リリースされる macOS のバージョンによっては、必ずダイアログが表示されるように修正されている可能性があります
:::
