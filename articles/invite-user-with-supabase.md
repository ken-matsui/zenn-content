---
title: "Supabase を使ってユーザー招待を実装する"
emoji: "📬"
type: "tech"
topics: ["supabase", "nextjs", "postgresql", "phoenix", "firebase"]
published: true
---

# 使うもの

* Supabase

https://supabase.com/

* Next.js

https://nextjs.org/

* `supabase-js`
  ```bash
  $ npm i @supabase/supabase-js
  ```

https://github.com/supabase/supabase-js

* `supabase-auth-helpers`
  ```bash
  $ npm i @supabase/supabase-auth-helpers
  ```

https://github.com/supabase-community/supabase-auth-helpers

# 実装方法

Supabase のユーザー招待には、`inviteUserByEmail` を使用します。

https://supabase.com/docs/reference/javascript/auth-api-inviteuserbyemail

## `service_role` キーについて

Supabase Client のドキュメントにある、`Auth (Server only)` と書かれた欄にあるユーザー招待等の関数は全て、Supabase の `service_role` キーが必要になります。
取得するには、Supabase Dashboard から、以下の通りに進み、`Reveal` をクリックした後に `Copy` をクリックします。

* Settings > API

![API Keys](/images/invite-user-with-supabase/api-keys.png)

`anon` キーは、ユーザーのログイン情報を用いて、RLS を介してデータを操作するためのキーです。
それに対して `service_role` キーは、いわば特権キーで、RLS を迂回する等の、ユーザー単位ではできない処理を行うためのキーになります。

そのため、現状 `anon` キーは、`NEXT_PUBLIC_SUPABASE_ANON_KEY` 等で、[ブラウザ側にも環境変数を渡している](https://nextjs.org/docs/basic-features/environment-variables#exposing-environment-variables-to-the-browser)と思います。
しかし、`service_role` キーは特権キーにあたるため、必ずブラウザ側に渡してはいけません。

つまり、実装する場合には、以下のような流れで実装することになります。

1. `service_role` キーをブラウザ側に渡さないように、`NEXT_PUBLIC_` プレフィックスを付けずに環境変数を定義する
2. そのキーを使うために、[API Route](https://nextjs.org/docs/api-routes/introduction) を実装し、そこに招待処理を実装する

## `service_role` キーを設定する

`.env.local` ファイルに、`SUPABASE_SERVICE_ROLE` として、先程コピーした内容をペーストします。

```bash
# Update these with your Supabase details from your project settings > API
NEXT_PUBLIC_SUPABASE_URL="https://your-project.supabase.co"
NEXT_PUBLIC_SUPABASE_ANON_KEY="your-anon-key"
SUPABASE_SERVICE_ROLE="your-service-role"  # This key has the ability to bypass Row Level Security. Never share it publicly.
```

## API Route を実装する

`inviteUserByEmail` を使用するため、招待には、email address が必要になります。
そのため、API Route を以下のように実装するのが綺麗かと思います。

`pages/api/invite/[email].ts`

```ts
import { withAuthRequired } from "@supabase/supabase-auth-helpers/nextjs";
import { createClient } from "@supabase/supabase-js";

export default withAuthRequired(async function InviteUser(req, res) {
    const { email } = req.query;
    const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL;
    const supabaseServiceRole = process.env.SUPABASE_SERVICE_ROLE;

    if (typeof email === "string" && supabaseUrl && supabaseServiceRole) {
        const supabaseAdmin = createClient(supabaseUrl, supabaseServiceRole);
        const { data, error } = await supabaseAdmin
            .auth
            .api
            .inviteUserByEmail(email);
        if (error) {
            res.status(error.status).json(error);
        } else {
            res.status(200).json(data);
        }
    } else {
        res.status(500).json({
            "error": "unknown error occurred",
        });
    }
});
```

コードが少し長めにも見えますが、やっていることは非常に簡単です。

1. ```ts
   export default withAuthRequired(async function InviteUser(req, res) {
   ```
   `supabase-auth-helpers` の `withAuthRequired` は、認証を強制します。そのため、認証していない状態でこの API を叩いてもエラーになります。これは、ログインしていないユーザーが他のユーザーを招待することなどありえないからです。
2. ```ts
   const { email } = req.query;
   ```
   [Dynamic API Routes](https://nextjs.org/docs/api-routes/dynamic-api-routes) を使用して、クエリから email address を取得するようにしています。
3. ```ts
   const supabaseAdmin = createClient(supabaseUrl, supabaseServiceRole);
   ```
   `supabase-js` の `createClient` 関数を使用します。いつもであれば、第 2 引数は `anon` キーになりますが、今回は `Auth (Server Only)` 欄の関数を使用したいため、`service_role` キーを使用します。
4. CORS 設定をしない
   CORS に関して何も指定しないということは、[同一オリジンのみを許可する設定](https://nextjs.org/docs/api-routes/introduction#caveats)となります。これは、Next.js のフロント側からしか叩くことが無いからです。
5. ```ts
   const { data, error } = await supabaseAdmin
        .auth
        .api
        .inviteUserByEmail(email);
    ```
    最後に、3 で作成した Admin Client を使用して、ユーザーを招待します。

# ブラウザ側から招待 API を叩く

後はブラウザ側から、API を叩くだけです。
今回では、シンプルな `fetch` を使用しますが、[`SWR`](https://swr.vercel.app/) 等を組み合わせることも可能かと思います。

以下の手順で、`pages/index.tsx` 等に実装していきます。

1. 招待関数を実装する
   ```ts
   const inviteUser = useCallback(async () => {
       await fetch(`/api/invite/${inputEmail}`);
   }, [inputEmail]);
   ```
2. Input にメールアドレスを入力できるようにする
   ```tsx
   const [inputEmail, setInputEmail] = useState<string>("");

   const handleChange = useCallback((event: ChangeEvent<HTMLInputElement>) => setInputEmail(event.target.value), []);

   return (
       <Input value={inputEmail} onChange={handleChange} />
   );
   ```
3. Button の onClick ハンドラで、1 で作成した招待関数を実行する
   ```tsx
   <Button onClick={inviteUser}>
       Invite
   </Button>
   ```

<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->
:::details フルコードはこちら
<!-- textlint-enable -->
```tsx
import type { ChangeEvent } from "react";
import { useCallback, useState } from "react";
import { Button, Input, InputGroup, InputRightElement } from "@chakra-ui/react";

function Invitation(): JSX.Element {
    const [inputEmail, setInputEmail] = useState<string>("");

    const handleChange = useCallback((event: ChangeEvent<HTMLInputElement>) => setInputEmail(event.target.value), []);

    const inviteUser = useCallback(async () => {
        await fetch(`/api/invite/${inputEmail}`);
    }, [inputEmail]);

    return (
        <InputGroup size="md">
            <Input
                pr="4.5rem"
                placeholder="Enter an email address to invite a user"
                value={inputEmail}
                onChange={handleChange}
            />
            <InputRightElement width="4.5rem">
                <Button isLoading={isLoading} h="1.75rem" size="sm" onClick={inviteUser}>
                    Invite
                </Button>
            </InputRightElement>
        </InputGroup>
    );
}
```
:::

上記を `next dev` で実行後、入力したメールアドレスに対して以下の画像のような招待メールが届けば、招待が上手く処理されています。

![Invitation Email](/images/invite-user-with-supabase/invitation-email.png)

# カスタムメールドメイン

今回の場合だと、`noreply@mail.app.supabase.io` からメールが届いていますが、以下の設定を変更することでカスタムドメインからメールが送信できるようです。

* Authentication > Settings

![Custom Email Domain](/images/invite-user-with-supabase/custom-email-domain.png)

https://www.reddit.com/r/Supabase/comments/rfvw42/configure_supabase_to_send_emails_from_my_domain/

# メールテンプレート

今回は、招待メールをデフォルトのまま使用しています。
テンプレートを編集したい場合は、以下のページにアクセスすることで編集できます。

* Authentication > Templates

![Email Template](/images/invite-user-with-supabase/email-template.png)

# [supabaseServerClient](https://supabase-community.github.io/supabase-auth-helpers/modules/nextjs_utils_supabaseServerClient.html) に関して

`supabase-auth-helpers` には、[`supabaseClient`](https://supabase-community.github.io/supabase-auth-helpers/modules/nextjs.html#supabaseClient) が実装されており、そのコンストラクタが自動で `NEXT_PUBLIC_SUPABASE_URL` と `NEXT_PUBLIC_SUPABASE_ANON_KEY` を読み込むように実装されています。
そのため、同パッケージの `supabaseServerClient` は、その名称から `SUPABASE_SERVICE_ROLE` を自動で読み込むように実装されているように感じますが、そうではありません。
こちらは、ブラウザ側における認証情報を、API 側に持ち越すためのクライアントになるため、今回のような `Auth (Server Only)` 欄の関数を使用する場合には権限が足りません。

こちらを使用する時は、API 側でユーザーの認証情報を用いて RLS を介しながら、何か処理を行いたい場合に使いましょう。
使い方に関しては、`supabase-auth-helpers` の `examples` ディレクトリが参考になります。

https://github.com/supabase-community/supabase-auth-helpers/blob/7a275700b0db392c43b3e381ec801cc779a4dd54/examples/nextjs/pages/api/protected-route.ts#L7-L13

# 招待直後のユーザー情報

招待した時点で、そのメールアドレスに紐づくユーザーテーブルが作成され、それに対して UUID が割り振られます。

![Invited User](/images/invite-user-with-supabase/invited-user.png)

そのため、例えば以下のテーブル構造の場合、招待処理の直後に、招待されたユーザーの UUID を `members` テーブルに Insert しておくことができます。
そうすることで、招待されたユーザーは、すぐに招待されたチームにアクセスできます。

```sql
create table teams (
  id serial primary key,
  name text
);

-- 2. Create many to many join
create table members (
  team_id bigint references teams,
  user_id uuid references auth.users
);

-- 3. Enable RLS
alter table teams
  enable row level security;

-- 4. Create Policy
create policy "Team members can update team details if they belong to the team."
  on teams
  for update using (
    auth.uid() in (
      select user_id from members
      where team_id = id
    )
  );
```

*Source: [Row Level Security - Supabase](https://supabase.com/docs/guides/auth/row-level-security#policies-with-joins)*

# 最後に

Supabase は非常に多くのことを簡単に実装でき、それでいてバックエンドが PostgreSQL なことを活かして、非常に柔軟な操作ができます。
そういったこともあり、Firebase と違い、技術的負債になることが少ない、安心できる便利なサービスかと思います。

さらに、Supabase 自体の開発速度が非常に早いことも利点として挙げられます。
先日には、[Presence](https://hexdocs.pm/phoenix/Phoenix.Presence.html) や [Socket.Broadcast](https://hexdocs.pm/phoenix/Phoenix.Socket.Broadcast.html) といった、[Phoenix Framework](https://www.phoenixframework.org/) の機能を活用した Realtime 機能をクローズドとしてですが、リリースされています。

https://supabase.com/blog/2022/04/01/supabase-realtime-with-multiplayer-features

初期は無料で使用できるので、色々と試してみてください。
