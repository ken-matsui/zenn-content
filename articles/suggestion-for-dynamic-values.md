---
title: "動的な値も `Did you mean 'hoge'?` したい"
emoji: "😎"
type: "tech"
topics: ["rust"]
published: true
---

[clap-rs](https://github.com/clap-rs/clap) には、suggestion というユーザーの typo を検知するための feature が実装されています。

https://github.com/clap-rs/clap#default-features

しかし、これは Runtime に定まる値に対して、typo を検知できません。
あくまで、事前に定義しておいたオプションリストに対して suggestion を提供しているだけです。

例えば [Zellij](https://github.com/zellij-org/zellij) の [v0.21.0](https://github.com/zellij-org/zellij/releases/tag/v0.21.0) より前のバージョンでは、存在しないセッションに対して attach しようとした時、ただ単にそのセッションが存在しない旨を伝えてくるだけでした。

```bash
$ zellij attach organic-snace
No session named "organic-snace" found.
```

合っていると思いながら、`zellij list-sessions` を実行すると、typo していたみたいなことが起こります。

```bash
$ zellij list-sessions
organic-snake
black-and-white-thing
draconian-flowers
```

そういったユースケースのために、suggestion という crate を開発しました。

https://github.com/ken-matsui/suggest

こちらを使用すると、以下のように Runtime に確定する値リストに対して、suggestion を提供できます。

```bash
$ zellij attach organic-snace
No session named "organic-snace" found.
  help: Did you mean `organic-snake`?
```

https://github.com/zellij-org/zellij/pull/843

# 使い方

使い方は非常にシンプルで、以下のように使用できます。

```rust
use suggestion::Suggest;

fn main() {
    let input = "instakk";

    let list_commands = vec!["update", "install"];
    if list_commands.contains(&input) {
        return;
    }

    if let Some(sugg) = list_commands.suggest(input) {
        println!("No command named `{}` found.", input);
        println!("Did you mean `{}`?", sugg);
    }
}
```

基本的な Primitive Types に対して、Trait を定義しているため、`use suggestion::Suggest` とすれば、`suggest` メソッドが生えてきます。

https://github.com/ken-matsui/suggest#supported-types

例えば、Zellij の場合だと、`list-sessions` サブコマンド用の関数が用意されているので、それを併用することで以下のように実装しています。

```rust
println!("No session named {:?} found.", name);
if let Some(sugg) = get_sessions().unwrap().suggest(name) {
    println!("  help: Did you mean `{}`?", sugg);
}
```

https://github.com/zellij-org/zellij/pull/843/files

# CLI として使用する

suggestion は、CLI も実装しています。

以下のコマンドでインストールできます。

```bash
$ cargo install suggestion
```

こちらの使い方もシンプルで、最初の引数が typo かもしれない入力で、その後ろの引数全てが suggestion したい値リストとなります。

```bash
$ suggest instakk update install
The `instakk` input is similar to `install`.
```

以下の README.md に詳細が載っているので、興味がある方は見ていただけると幸いです。

https://github.com/ken-matsui/suggest#cli

# suggestion 自体の実装に関して

suggestion 自体の実装に関しても、興味のある方向けに紹介いたします。

## lev_distance

https://github.com/ken-matsui/lev_distance

[Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance) というアルゴリズムを実装している crate です。

簡単にいうと、文字列の近さを計測するためのアルゴリズムです。

Rust のコンパイラが、typo を検知するために内部的に使用していたソースコードを、一般的な String でも使用できるように改変したものになります。

https://github.com/rust-lang/rust/blob/0fb1c371d4a14f9ce7a721d8aea683a6e6774f6c/compiler/rustc_span/src/lev_distance.rs

Rust のコンパイラ専用ということで、実装には Symbol が使用されていますが、それをそのまま切り出すわけにはいかないため、String を使用するように置き換えています。

https://github.com/rust-lang/rust/blob/d6082292a6f3207cbdacd6633a5b9d1476bb6772/compiler/rustc_span/src/symbol.rs#L1625

上記の crate は、Rust のコンパイラとして指定されているライセンスのもとで公開されています。
ありがとうございます。

https://github.com/ken-matsui/lev_distance#license

## suggestion

https://github.com/ken-matsui/suggest

後は、上記の `lev_distance` crate を使用して、使用しやすいインターフェースを実装しているだけとなっています。

https://github.com/ken-matsui/suggest/blob/main/src/lib.rs

# 最後に

suggestion は、CLI アプリケーションを実装しているとどうしてもどこかで欲しくなる機能ですが、実装が面倒だと感じていました。
それを今回 crate にすることで、簡単に再利用できるようにしました。

色々な方が、CLI アプリケーションを実装したことがあると思うので、この crate が役に立てば幸いです。
