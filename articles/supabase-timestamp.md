---
title: "Supabase の時間関係のカラムに関して"
emoji: "⏱"
type: "tech"
topics: ["supabase", "timestamp"]
published: true
---

よくサービスを組んでいると、大概のテーブルに `created_at` と `updated_at` のカラムを用意していると思います。
命名は違っても、同じ意味のカラムを使用することが多いかと思います。

そこで、そういったカラムを Supabase で実現する方法に関して紹介させていただきます。

# created_at

こちらは非常にシンプルに表現できます。

## Supabase Console

https://github.com/supabase/supabase/issues/379#issuecomment-1001994125

Supabase のコンソールでは、テーブルを作成する時に自動で追加されるようになっています。

![Created at](/images/supabase-timestamp/created-at.png)

細かいところも追っていくと、`timestampz` タイプというのは、タイムゾーンを含むタイムスタンプです。
タイムゾーンを含まなくて良い場合は、`timestamp` タイプが適しています。

Default Value としては、`NOW()` が指定されています。
こちらは、`now()` と小文字にしても問題が無いのと、`timestamp` と `timestampz` タイプの両方で問題なく動作します。

Default Value なので、そのカラムが作成されたタイミングで値を指定していなければ、その時の時間が自動で挿入されます。

## SQL Editor

SQL Editor から直接いじりたい場合は、以下の SQL クエリを叩くと `created_at` カラムを作成できます。

```sql
create table public.tests (
    created_at timestampz not null default now()
);
```

# updated_at

こちらは少しややこしいです。

`moddatetime` という Extension を使用することで実現できます。

## Supabase Console

Supabase のコンソールからの場合は、`created_at` の時と同様に `updated_at` とだけ変更してテーブルを作成してください。

![Updated at](/images/supabase-timestamp/updated-at.png)

次に、`moddatetime` 拡張を有効にするために、Database > Extensions へ移動し、`moddatetime` を検索すると以下の画像のようになります。

![moddatetime](/images/supabase-timestamp/moddatetime.png)

後は、トグルボタンをクリックし、`Confirm` を押すと、`moddatetime` 拡張が有効になります。

最後に、Trigger を作成するため、Database > Triggers へ移動し、以下の画像のように入力します。

![Trigger](/images/supabase-timestamp/trigger.png)

後は、`Confirm` をボタンをクリックすれば完了です。

## SQL Editor

https://github.com/supabase/supabase/issues/379#issuecomment-755289862

SQL Editor から直接いじりたい場合は、以下の SQL クエリを叩くと `updated_at` カラムを作成できます。

```sql
create table public.tests (
    updated_at timestampz not null default now()
);
```

次に、`moddatetime` 拡張を有効にします。

```sql
create extension if not exists moddatetime schema extensions;
```

そして、Trigger を設定すれば完了です。

```sql
create trigger handle_updated_at before update on public.tests
  for each row execute procedure moddatetime (updated_at);
```

# 最後に

時間関係のカラムは非常に多く使うので、もっとシンプルになると嬉しいですね。

殆どのカラムに `updated_at` を付ける構成になると、Trigger の発行が大変そうです。

こちらの記事が参考になれば幸いです。
