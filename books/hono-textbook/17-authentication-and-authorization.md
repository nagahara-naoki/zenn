---
title: "Web APIの認証と認可"
---

前章では、タスク一覧APIに検索、並び替え、ページネーションを追加しました。

APIとして、かなり実用的になってきました。
しかし、まだ大きな問題があります。

今のAPIは、誰でもすべてのタスクを操作できます。

```txt
誰でもタスクを作れる
誰でもタスクを見られる
誰でもタスクを更新できる
誰でもタスクを削除できる
```

学習用なら問題ありません。
でも、実際のアプリケーションでは危険です。

この章では、認証と認可の考え方を整理します。
次章では、その考え方をもとにJWT認証を実装します。

## 認証と認可の違い

認証と認可は、よく混同されます。

言葉は似ていますが、役割は違います。

| 用語 | 英語 | 意味 |
| --- | --- | --- |
| 認証 | Authentication | あなたは誰ですか |
| 認可 | Authorization | あなたはそれをしてよいですか |

たとえば、会社のオフィスを想像してください。

```mermaid
flowchart LR
  A["社員証を見せる"] --> B["本人確認"]
  B --> C["認証"]
  D["会議室に入る"] --> E["権限確認"]
  E --> F["認可"]
```

社員証で本人確認するのが認証です。

その人が特定の会議室に入ってよいかを確認するのが認可です。

Web APIでも同じです。

```txt
認証: このリクエストは user_123 から来た
認可: user_123 は task_abc を更新してよい
```

## 認証だけでは足りない

ログインしているユーザーなら、何でもできるわけではありません。

たとえば、タスク管理APIでは次のような制御が必要です。

- 自分のタスクだけを見られる
- 自分のタスクだけを更新できる
- 自分のタスクだけを削除できる
- 管理者だけが全ユーザーのタスクを見られる

これは認可の話です。

認証は入口です。
認可は操作ごとの確認です。

```mermaid
flowchart TB
  A["Request"] --> B{"ログイン済み？"}
  B -- "No" --> C["401 Unauthorized"]
  B -- "Yes" --> D{"操作してよい？"}
  D -- "No" --> E["403 Forbidden"]
  D -- "Yes" --> F["処理を実行"]
```

この流れを区別しておくと、APIのエラー設計もわかりやすくなります。

## IDとパスワード

もっとも基本的な認証は、IDとパスワードです。

```txt
email: alice@example.com
password: ********
```

ユーザーはログイン時に、メールアドレスとパスワードを送ります。

サーバーは、保存されているユーザー情報と照合します。

```mermaid
sequenceDiagram
  participant Client
  participant API
  participant DB

  Client->>API: email / password
  API->>DB: emailでユーザーを検索
  DB-->>API: user
  API->>API: パスワードを検証
  API-->>Client: token
```

ここで絶対に守るべきことがあります。

パスワードを平文で保存してはいけません。

## ハッシュと暗号化の違い

パスワードを保存するときは、暗号化ではなくハッシュ化します。

この違いは大切です。

| 種類 | 戻せるか | 主な用途 |
| --- | --- | --- |
| 暗号化 | 戻せる | 秘密情報をあとで復元したい |
| ハッシュ化 | 戻せない | パスワード検証 |

パスワードは、あとで復元する必要がありません。

ログイン時に入力されたパスワードを同じ方法でハッシュ化し、保存済みのハッシュと比較できれば十分です。

```txt
入力されたパスワード
↓
ハッシュ化
↓
保存済みハッシュと比較
```

サーバー側でも、元のパスワードを知らない状態にしておくのが基本です。

## パスワードを平文で保存しない

悪い例です。

```json
{
  "email": "alice@example.com",
  "password": "password123"
}
```

この状態でデータベースが漏えいすると、ユーザーのパスワードがそのまま流出します。

良い例です。

```json
{
  "email": "alice@example.com",
  "passwordHash": "$2b$10$..."
}
```

保存するのは、パスワードそのものではなくハッシュです。

実務では、bcrypt、Argon2、scryptなど、パスワード保存向けのアルゴリズムを使います。

単純なSHA-256だけでパスワードを保存するのは避けます。
高速すぎるハッシュは、総当たり攻撃にも高速に試されてしまうからです。

## Cookie認証とセッション認証

Webアプリケーションでよく使われるのが、Cookieとセッションを使う認証です。

流れは次のようになります。

```mermaid
sequenceDiagram
  participant Browser
  participant API
  participant Store

  Browser->>API: login
  API->>Store: sessionを保存
  API-->>Browser: Set-Cookie: session_id=...
  Browser->>API: Cookie: session_id=...
  API->>Store: sessionを確認
  API-->>Browser: protected data
```

サーバー側にセッションを保存し、ブラウザにはセッションIDをCookieで持たせます。

この方式の利点は、サーバー側でセッションを無効化しやすいことです。

一方で、APIサーバーが複数ある場合は、セッションストアを共有する必要があります。

## Bearer Token

APIでは、`Authorization` ヘッダーにトークンを入れる方式もよく使われます。

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

`Bearer` は「このトークンを持っている人」という意味合いです。

HonoにはBearer Tokenを検証するMiddlewareがあります。
固定のAPIトークンを守るような用途では、`bearerAuth()` が使えます。

```ts
import { bearerAuth } from 'hono/bearer-auth'

const adminToken = 'admin-secret-token'

app.use('/admin/*', bearerAuth({ token: adminToken }))
```

ただし、ユーザーごとのログインには、固定トークンだけでは足りません。

そこで、次章ではJWTを使います。

## JWTの構造

JWTは、JSON Web Tokenの略です。

JWTは、主に3つの部分からできています。

```txt
header.payload.signature
```

図にすると、こうです。

```mermaid
flowchart LR
  A["Header"] --> D["JWT"]
  B["Payload"] --> D
  C["Signature"] --> D
```

| 部分 | 内容 |
| --- | --- |
| Header | 署名アルゴリズムなど |
| Payload | ユーザーIDや有効期限など |
| Signature | 改ざんされていないことを確認する署名 |

Payloadには、たとえば次のような情報を入れます。

```json
{
  "sub": "user_123",
  "role": "user",
  "iss": "hono-task-api",
  "aud": "hono-task-client",
  "exp": 1784217600
}
```

`sub` はSubjectです。
多くの場合、ユーザーIDを入れます。

`exp` は有効期限です。
JWTでは、有効期限を短めにすることが重要です。

## JWTは暗号化ではない

JWTについて、特に大切な注意があります。

JWTのPayloadは、基本的に読めます。

署名されているので改ざんは検出できます。
しかし、暗号化されているわけではありません。

そのため、JWTのPayloadに次のような情報を入れてはいけません。

- パスワード
- クレジットカード番号
- 個人番号
- 秘密鍵
- 外部サービスのAPIキー

JWTに入れるのは、クライアントから見えても問題ない最小限の情報にします。

## Access TokenとRefresh Token

JWT認証では、Access TokenとRefresh Tokenを分けることがあります。

| 種類 | 役割 | 有効期限 |
| --- | --- | --- |
| Access Token | API呼び出しに使う | 短い |
| Refresh Token | Access Tokenを再発行する | 長い |

Access Tokenだけを長期間有効にすると、漏れたときの被害が大きくなります。

そこで、Access Tokenは短命にします。
期限が切れたら、Refresh Tokenで新しいAccess Tokenを発行します。

```mermaid
sequenceDiagram
  participant Client
  participant API

  Client->>API: Access TokenでAPI呼び出し
  API-->>Client: 401 Token expired
  Client->>API: Refresh Tokenで再発行
  API-->>Client: New Access Token
```

本書では、まずAccess Tokenを使ったシンプルなJWT認証を実装します。

Refresh Tokenは、ログアウトや長期ログインと一緒に考えると理解しやすいため、次章で考え方だけ触れます。

## CookieとAuthorizationヘッダーの選び方

トークンをどこに持たせるかも重要です。

代表的には、CookieとAuthorizationヘッダーがあります。

| 方式 | 向いている場面 | 注意点 |
| --- | --- | --- |
| Cookie | ブラウザ中心のWebアプリ | CSRF対策が必要 |
| Authorizationヘッダー | SPA、モバイルアプリ、外部API | トークン保管に注意 |

Authorizationヘッダーを使う場合、クライアントは毎回ヘッダーを付けます。

```http
Authorization: Bearer <access_token>
```

Cookieを使う場合、ブラウザが自動でCookieを送ります。
そのため、CSRF対策を考える必要があります。

どちらが絶対に正しい、という話ではありません。
アプリケーションの種類、クライアント、セキュリティ要件で選びます。

## ステートレス認証の利点と弱点

JWTは、よくステートレス認証として使われます。

サーバーは、トークンの署名を検証できれば、セッションストアへ問い合わせずにユーザーを判断できます。

```mermaid
flowchart LR
  A["Request + JWT"] --> B["署名を検証"]
  B --> C["Payloadを読む"]
  C --> D["userIdを取得"]
```

利点は、サーバー側の状態を少なくできることです。

一方で、弱点もあります。

| 利点 | 弱点 |
| --- | --- |
| セッションストアが不要になりやすい | 発行済みトークンを即時失効しにくい |
| 複数サーバーで扱いやすい | 漏えい時の影響を考える必要がある |
| 外部クライアントと相性がよい | Payloadに情報を入れすぎやすい |

JWTを使えば自動的に安全になるわけではありません。

有効期限、署名鍵、保存場所、失効方法まで含めて設計します。

## 認証方式を選ぶ基準

認証方式を選ぶときは、次の観点で考えます。

| 観点 | 確認すること |
| --- | --- |
| クライアント | ブラウザ、モバイル、外部システムのどれか |
| ログアウト | 即時ログアウトが必要か |
| トークン更新 | 長期ログインが必要か |
| セキュリティ | CSRF、XSS、漏えい時の影響 |
| 運用 | セッションストアや鍵管理をどうするか |

タスク管理APIは、SPAや外部クライアントから呼び出すAPIとして設計します。

そのため、本書では次の方式を採用します。

```txt
Authorization: Bearer <JWT>
```

## 本書で実装する認証方式

本書では、次章で次の流れを実装します。

```mermaid
sequenceDiagram
  participant Client
  participant API
  participant DB

  Client->>API: POST /auth/register
  API->>DB: ユーザー作成
  API-->>Client: user

  Client->>API: POST /auth/login
  API->>DB: ユーザー検索
  API->>API: パスワード検証
  API-->>Client: accessToken

  Client->>API: GET /tasks + Bearer token
  API->>API: JWT検証
  API->>DB: 自分のタスクだけ取得
  API-->>Client: tasks
```

実装するAPIは、次のとおりです。

| メソッド | パス | 説明 |
| --- | --- | --- |
| `POST` | `/auth/register` | ユーザー登録 |
| `POST` | `/auth/login` | ログイン |
| `GET` | `/tasks` | 自分のタスク一覧 |
| `POST` | `/tasks` | 自分のタスク作成 |
| `PATCH` | `/tasks/:id` | 自分のタスク更新 |
| `DELETE` | `/tasks/:id` | 自分のタスク削除 |

「自分のタスクだけ」という制約が、認可です。

## 401と403の違い

認証と認可を分けると、ステータスコードも整理できます。

| ステータス | 意味 | 例 |
| --- | --- | --- |
| `401 Unauthorized` | 認証されていない | トークンがない、無効、期限切れ |
| `403 Forbidden` | 認証済みだが権限がない | 他人のタスクを更新しようとした |

名前だけ見ると、`401 Unauthorized` が「認可されていない」に見えるので少し紛らわしいです。

実務では、次のように覚えるとよいです。

```txt
401: まずログインしてください
403: ログイン済みですが、その操作はできません
```

タスク管理APIでは、未ログインなら401。
他人のタスクを操作しようとしたら403、または存在を隠したい場合は404を返す設計もあります。

どちらにするかは、APIの性質で決めます。

## まとめ

この章では、Web APIの認証と認可を整理しました。

- 認証は「あなたは誰か」を確認すること
- 認可は「その操作をしてよいか」を確認すること
- パスワードは平文で保存しない
- パスワード保存には、パスワード向けのハッシュを使う
- Bearer TokenはAPIでよく使われる
- JWTは署名されているが、Payloadは読める
- Access Tokenは短命にする
- CookieとAuthorizationヘッダーは、アプリの性質で選ぶ
- 401と403を使い分ける

次章では、実際にJWT認証を実装します。

ユーザー登録、ログイン、JWT発行、JWT検証、そして「自分のタスクだけ操作できる」認可まで進めます。
