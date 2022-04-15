---
title: "Supabase Auth のユーザー情報を操作する方法"
emoji: "👪"
type: "tech"
topics: ["supabase", "postgresql"]
published: true
---

Supabase Auth は非常に便利で、[RLS](https://supabase.com/docs/guides/auth/row-level-security) を組み合わせることで、制限した内容のみを Select したりできます。

https://supabase.com/docs/guides/auth

そういった認証周りをよしなにしてくれる点は非常に便利ではあるのですが、例えばユーザーを検索できるようにしたい時、Supabase Auth のテーブルをユーザーから参照できません。
`Anon` キーを経由している以上、`public.*` と、`create schema` クエリで作成したスキーマ内のテーブルしか Select 等の操作ができないようになっています。
それ以外の Supabase が自動で生成したスキーマ内のテーブルは、あらゆる操作が Supabase Client からはできません。

そこで Supabase Triggers と Supabase Database Functions を組み合わせることで、`auth.users` テーブルに Insert されたら、`public.users` テーブルにも同一のユーザーを作成する、という処理を実装します。

https://supabase.com/blog/2021/07/30/supabase-functions-updates

# `public.users` テーブルを作成する

```sql
create table public.users (
  id uuid primary key references auth.users on delete cascade,
  email text not null,
  first_name text,
  last_name text
);
```

Foreign Key として `auth.users` を指定し、`on delete cascade` とすることで、ユーザーの削除時にこちらのテーブルからも自動で削除できます。

https://supabase.com/docs/reference/javascript/auth-api-deleteuser

また PostgreSQL において、`no action` がデフォルトとして設定されているため、明示的に `cascade` として設定する必要があります。

https://stackoverflow.com/a/49677327

# RLS を設定する

```sql
alter table public.users
  enable row level security;

create policy "Allow select for all authenticated users."
  on public.users for select using (
    auth.role() = 'authenticated'
  );

create policy "Allow update for users themselves."
  on public.users for update using (
    auth.uid() = id
  );
```

認証済みユーザー全てからユーザー一覧を見られるようにしたいため、`auth.role() = 'authenticated'` と設定します。
また、ユーザー自身から `first_name` や `last_name` を設定できるようにしたいため、その時の認証ユーザーと編集対象の `id` が一致するかどうかをチェックするように設定します。

こういった設定は、サービスの性質に合わせて変更する必要があります。

# `auth.users` をコピーする Database Functions を作成する

```sql
create function public.handle_new_user()
returns trigger as $$
language plpgsql
security definer
set search_path = public
begin
  insert into public.users (id, email)
  values (new.id, new.email);
  return new;
end;
$$;
```

後は、トリガー用の PostgreSQL Function を作成します。
`security definer` は、その関数作成者の権限で、RLS を回避してこの関数を実行するために設定しています。
ユーザーが叩く可能性のある関数に対しては、`security invoker` を使って、関数使用者の権限で関数が実行されるように設定することをおすすめします。

https://supabase.com/docs/guides/auth/row-level-security#policies-with-security-definer-functions

https://qiita.com/kabochapo/items/26b1bb753116a6904664#infinite-recursion

PostgreSQL では `invoker` がデフォルトとして設定されており、その場合は、RLS に基づいて関数を実行できます。

https://www.postgresql.org/docs/current/sql-createfunction.html

# `auth.users` の Insert をトリガーにする

```sql
create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();
```

これで、`auth.users` に Insert が走る、すなわち新しいユーザーが Sign up する度に、`public.users` にコピーされます。

> 上記のクエリは、こちらを参考にしました。

https://supabase.com/docs/guides/auth/managing-user-data#advanced-techniques

# `anon` キーで `public.users` をクエリする

後は、以下のようにシンプルな方法で `public.users` をクエリできます。

```ts
import { createClient } from '@supabase/supabase-js'

// Create a single supabase client for interacting with your database
const supabaseClient = createClient('https://xyzcompany.supabase.co', 'public-anon-key')

const { data, error } = await supabaseClient
    .from<User>("users")
    .select("*");
```

# 最後に

どうしてもコピーになってしまうため、情報をどこまで持ってくるのか、情報を新鮮に保つ方法などが重要になってきます。

例えば、Google 認証を使用する際は、フルネームが情報に入ってくるため、そちらを使用できます。
ただし、Google 認証以外の認証方法を使用する場合は、それぞれどこに情報が入っているのかを把握する必要があります。
そのため、現実的な解決策として、`id` と `email` のみコピーするという対応策に帰着しました。

また、メールアドレスを変更した時、`public.users` も追従する必要があります。

https://supabase.com/docs/reference/javascript/auth-update

その時は、`after update on auth.users` としてトリガーと関数を作成する必要があるかと思います。

色々と扱いづらい点もありますが、セキュリティを担保するためにこのような実装となっているため、どうしても仕方無いところはあるかなと思います。

この記事が参考になれば、幸いです。
