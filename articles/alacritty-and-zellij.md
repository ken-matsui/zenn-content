---
title: "Alacritty 起動時に Zellij を自動で起動する"
emoji: "💻"
type: "tech"
topics: ["terminal", "alacritty", "zellij", "tmux"]
published: true
---

# tmux と Zellij の違い

alacritty を使用されている方は、[それ自体には tab 等の機能が無い](https://github.com/alacritty/alacritty/blob/673710487afac8596a9f18fea9e04aeada32c2be/README.md?plain=1#L95-L101)ため、tmux と合わせて使っている方が多いと思います。
公式から、tmux へのリンクが貼られているぐらいです。

しかし、tmux 使っていたときは、自分にとって使い心地良くするために多くの設定ファイルを用意し、Tmux Plugin Manager と組み合わせる等の作業が必要で大変でした。
一方で [Zellij](https://github.com/zellij-org/zellij) では、直感的だと思うキーマップを変更するぐらいで、yaml で記述できることもあり、非常に設定が楽でした。

ターミナルマルチプレクサを真に活用できていないと言われればそれまでではありますが、基本的な画面分割と、タブがあれば充分という自分にとっては、Zellij が最適な選択でした。
また、下のスクリーンショットのように、常に使い方が下部に表示されているため、ターミナルマルチプレクサに慣れていない方にも使いやすいという利点もあります。
もちろん、tmux にしか存在しないニッチな機能を常用している方は、tmux を使う方が望ましいです。

![Screenshot of Zellij](/images/alacritty-and-zellij/zellij-preview.png)
*Screenshot of Zellij
Source: [Screenshots on zellij.dev](https://zellij.dev/screenshots/)*

Zellij は機能が少ないということは全く無く、非常に多くの機能が実装されており、開発が活発で、速度も早いです。
ただ、Alacritty とは違い、メンテナーの指針として速度に重きを置いていないので、今後は速度が遅くなる可能性はあります。
そのため、最速のターミナルセットを求めている場合は、tmux の方が適している可能性があります。

また、tmux との互換性リストが公開されており、tmux から Zellij へ移行しようと考えている場合は役に立つんじゃないかなと思います。

https://github.com/zellij-org/zellij/issues/376

Zellij 自体に関しては、以下の記事で詳しく説明されています。

https://zenn.dev/gosarami/articles/4becaa18273216fec5ee

# Alacritty からターミナルマルチプレクサを自動起動する

## tmux の場合

以下のように設定することで、Alacritty を起動する際に、tmux も合わせて自動起動できます。

`.config/alacritty/alacritty.yml`

```yaml
shell:
  program: /bin/zsh
  args:
    - -l
    - -c
    - "tmux a -t 0 || tmux"
```

参考：https://lon.sagisawa.me/2021/02/better-terminal-environment

上記を設定した上で Alacritty を起動すると、自動で以下の処理を行い、アタッチやセッションを作成してくれます。

1. tmux のセッションが 1 つでも残っていれば、それにアタッチする
2. セッションが 1 つも見つからなければ、新規でセッションを作成する

誤ってターミナルごと kill してしまった時に、そのセッションに戻れるため、便利な設定です。
それが不要な場合は、`tmux` のみを記述すると良いでしょう。

## Zellij の場合

Zellij の場合は、以下のように設定することで、Alacritty を起動する際に Zellij も合わせて自動起動できます。

[`.config/alacritty/alacritty.yml`](https://github.com/ken-matsui/dotfiles/blob/5426a08732a79711ccd77e573e6c781ed3a3e207/.config/alacritty/alacritty.yml#L133-L138)

```yaml
shell:
  program: /bin/zsh
  args:
    - -l
    - -c
    - "zellij attach --index 0 --create"
```

以下の PR で実装した機能で、上記の tmux の設定と全く同じ挙動をする設定になります。

https://github.com/zellij-org/zellij/pull/824

Zellij における `--create` オプションは、tmux における `|| tmux` と同じ意味で、もしアタッチ対象のセッションが存在しなければ、新しくセッションを作成します。

# 最後に

以上の簡単な設定で、Alacritty 起動時に、Zellij を自動で起動することが実現できました。
これは、Alacritty 用の設定のため、VS Code や Jetbrains 系 IDE 内のターミナルでは、Zellij が起動しません。
それらはそもそもタブが実装されていて、Zellij は不要だと思うので、現状の設定が非常に綺麗かなと思います。

大量の設定を弄ることが大変と感じる方や、キーバインディングを覚えるのが面倒な方は、Zellij が相性良いかもしれないので、是非試してみてください。

https://github.com/zellij-org/zellij
