---
title: "Hono RPCとHono Client"
---

前章では、JWT認証を実装しました。

これで、APIはかなり実用的になりました。

ただし、クライアント側からAPIを呼ぶときには、まだ面倒があります。

たとえば、次のようなコードです。

```ts
const res = await fetch('https://api.example.com/tasks', {
  headers: {
    Authorization: `Bearer ${accessToken}`,
  },
})

const data = await res.json()
```

この書き方では、`data` の型がわかりません。
パスやメソッドを間違えても、TypeScriptは気づけません。

Honoには、この問題を軽くする仕組みがあります。

それがHono RPCとHono Clientです。

## Hono RPCとは何か

Hono RPCは、サーバー側のルート定義から、クライアント側の型を推論する仕組みです。

```mermaid
flowchart LR
  A["Hono Server"] --> B["typeof app"]
  B --> C["AppType"]
  C --> D["Hono Client"]
  D --> E["型安全なAPI呼び出し"]
```

Hono公式では、RPCは「サーバーとクライアントの間でAPI仕様を共有する機能」と説明されています。

ポイントは、OpenAPIのように別の仕様ファイルを先に書くのではなく、Honoのルート定義そのものを型として共有することです。

## RPCという名前の意味

RPCはRemote Procedure Callの略です。

直訳すると「遠隔の手続きを呼び出す」です。

ただし、Hono RPCを使うからといって、REST APIをやめるわけではありません。

APIの実体は、これまで作ってきたHTTP APIです。

```txt
GET /tasks
POST /tasks
PATCH /tasks/:id
DELETE /tasks/:id
```

Hono RPCは、そのHTTP APIをクライアント側から型安全に呼び出すための道具です。

## AppType

Hono RPCで中心になるのが `AppType` です。

サーバー側で、Honoアプリケーションの型をexportします。

```ts
// src/index.ts
import { Hono } from 'hono'
import { authRoute } from './routes/auth'
import { tasksRoute } from './routes/tasks'

const app = new Hono()
  .route('/auth', authRoute)
  .route('/tasks', tasksRoute)

export type AppType = typeof app

export default app
```

この `AppType` が、API仕様の型になります。

クライアント側では、この型を読み込んでHono Clientを作ります。

ここで共有しているのは、実行時の`app`ではなく型だけです。
クライアントはサーバーの処理を直接呼ぶわけではありません。
あくまでHTTP APIを呼び出しますが、その呼び出し方をTypeScriptが補助してくれる、という理解が安全です。

## typeofでAPIの型を表現する

Honoでは、ルート定義をチェーンで書くほど型推論が効きやすくなります。

```ts
const tasksRoute = new Hono()
  .get('/', (c) => c.json({ tasks: [] }, 200))
  .post('/', (c) => c.json({ task: {} }, 201))
  .get('/:id', (c) => c.json({ task: {} }, 200))
```

この `tasksRoute` には、ルート情報が型として積み上がっています。

```mermaid
flowchart LR
  A["new Hono()"] --> B[".get('/')"]
  B --> C[".post('/')"]
  C --> D[".get('/:id')"]
  D --> E["typeof tasksRoute"]
```

第13章で扱ったように、ルート定義を関数の中へ隠しすぎると、型推論が弱くなることがあります。

Hono RPCを使うなら、ルート定義そのものを `const` として作り、その型をexportするのが基本です。

## hc()でClientを作る

クライアント側では、`hono/client` から `hc()` をimportします。

```ts
// src/client.ts
import { hc } from 'hono/client'
import type { AppType } from '../server'

export const client = hc<AppType>('http://localhost:8787')
```

これで、`client` からAPIを呼び出せます。

`hc<AppType>()`は、APIクライアントを作る入口です。
引数のURLは実際のAPIの場所で、型引数の`AppType`は「このAPIにはどんなルートがあるか」を表します。
URLと型の両方がそろって、初めて型付きのHTTPクライアントとして使えます。

```ts
const res = await client.tasks.$get()

if (res.ok) {
  const data = await res.json()
  console.log(data.tasks)
}
```

`client.tasks.$get()` は、`GET /tasks` に対応しています。

Hono Clientの戻り値は、通常の `fetch()` と同じように `Response` として扱えます。

```ts
const data = await res.json()
```

## 入力型とレスポンス型が推論される仕組み

Hono RPCでは、主に2つの型が推論されます。

| 推論されるもの | もとになるもの |
| --- | --- |
| 入力型 | Validator |
| レスポンス型 | Handlerの `c.json()` |

たとえば、タスク作成APIです。

```ts
export const createTaskSchema = z.object({
  title: z.string().min(1).max(100),
})

const tasksRoute = new Hono()
  .post('/', zValidator('json', createTaskSchema), async (c) => {
    const body = c.req.valid('json')

    return c.json(
      {
        task: {
          id: crypto.randomUUID(),
          title: body.title,
          completed: false,
        },
      },
      201,
    )
  })
```

クライアント側では、`json` に渡す値が推論されます。

```ts
const res = await client.tasks.$post({
  json: {
    title: 'Learn Hono RPC',
  },
})
```

`title` が足りない場合や、不要な型の値を渡した場合、TypeScriptが教えてくれます。

## Validatorが必要な理由

Hono RPCで入力型を推論するには、Validatorが重要です。

次のように、Handlerの中で手動で `c.req.json()` を読むだけだと、クライアント側へ入力型を伝えにくくなります。

```ts
// 型推論が弱くなりやすい
app.post('/tasks', async (c) => {
  const body = await c.req.json()
  return c.json({ task: body }, 201)
})
```

Validatorを使うと、APIの入口で入力の形が明確になります。

```ts
app.post('/tasks', zValidator('json', createTaskSchema), async (c) => {
  const body = c.req.valid('json')
  return c.json({ task: body }, 201)
})
```

これは、実行時の安全性だけでなく、クライアント側の型推論にも効きます。

## ルートチェーンが必要な理由

Hono RPCでは、ルートをチェーンで定義することが大切です。

```ts
const app = new Hono()
  .route('/auth', authRoute)
  .route('/tasks', tasksRoute)

export type AppType = typeof app
```

Hono公式の例でも、複数のアプリを組み合わせる場合は、`app.route()` にルート定義の戻り値を渡す形が紹介されています。

次のように、登録関数の中へ隠すと、実行時は動いても型推論が期待どおりにならないことがあります。

```ts
function registerRoutes(app: Hono) {
  app.route('/tasks', tasksRoute)
}

const app = new Hono()
registerRoutes(app)

export type AppType = typeof app
```

Hono RPCを前提にするなら、ルート定義の型が残る書き方を選びます。

## GETを呼び出す

タスク一覧を取得します。

```ts
const res = await client.tasks.$get({
  query: {
    limit: '20',
    offset: '0',
    completed: 'false',
    sort: 'createdAt',
    order: 'desc',
  },
})

if (res.ok) {
  const data = await res.json()
  console.log(data.tasks)
  console.log(data.meta.hasNext)
}
```

クエリパラメーターはURLに乗るため、文字列として送るのが自然です。

サーバー側では、Zodの `z.coerce.number()` などを使って数値へ変換します。

## POSTを呼び出す

タスク作成APIです。

ここでは、`fetch()`のようにURLや`method: 'POST'`を手で書きません。
`client.tasks.$post()`という呼び出し自体が、`POST /tasks`を表しています。
そのため、パスやメソッドの指定ミスを減らせます。

```ts
const res = await client.tasks.$post({
  json: {
    title: 'Write Hono textbook',
  },
})

if (res.status === 201) {
  const data = await res.json()
  console.log(data.task.id)
}
```

JSONボディは、`json` プロパティに渡します。

```ts
{
  json: {
    title: 'Write Hono textbook',
  }
}
```

Hono Clientは、この入力をサーバー側のValidatorから推論します。

## PATCHとDELETEを呼び出す

パスパラメーターがあるルートは、`param` に値を渡します。

```ts
const updateRes = await client.tasks[':id'].$patch({
  param: {
    id: 'task_123',
  },
  json: {
    completed: true,
  },
})
```

削除も同じです。

```ts
const deleteRes = await client.tasks[':id'].$delete({
  param: {
    id: 'task_123',
  },
})

if (deleteRes.status === 204) {
  console.log('deleted')
}
```

パス文字列を手で組み立てなくてよいので、入力ミスが減ります。

```ts
// 手書きだと間違えやすい
fetch(`/task/${id}`)
```

```ts
// Hono Clientならルート型に沿って呼び出せる
client.tasks[':id'].$delete({ param: { id } })
```

## Headerを送る

認証が必要なAPIでは、Authorizationヘッダーを送ります。

```ts
const res = await client.tasks.$get(
  {
    query: {
      limit: '20',
      offset: '0',
    },
  },
  {
    headers: {
      Authorization: `Bearer ${accessToken}`,
    },
  },
)
```

毎回ヘッダーを書くのが面倒な場合は、クライアントを作る場所でラップします。

```ts
export function createApiClient(accessToken: string) {
  return hc<AppType>('http://localhost:8787', {
    headers: {
      Authorization: `Bearer ${accessToken}`,
    },
  })
}
```

アプリケーションの構成によっては、リクエストごとにトークンを差し替えたいこともあります。
その場合は、呼び出し時にヘッダーを渡す形が扱いやすいです。

## res.okとエラーレスポンス

Hono Clientの戻り値は `Response` なので、`res.ok` が使えます。

```ts
const res = await client.tasks[':id'].$get({
  param: {
    id: 'task_123',
  },
})

if (!res.ok) {
  const error = await res.json()
  console.error(error.message)
  return
}

const data = await res.json()
console.log(data.task)
```

ステータスコードを `c.json()` で明示しておくと、クライアント側の型も扱いやすくなります。

```ts
return c.json({ message: 'Task not found' }, 404)
```

成功時も、ステータスコードを明示しておくと読みやすいです。

```ts
return c.json({ task }, 200)
```

Hono RPCでは、グローバルな `app.onError()` やMiddlewareのレスポンスが、自動ですべてのルートの型へ混ざるわけではありません。

そのため、クライアントで扱いたいエラーは、各ルートで明示しておくとわかりやすくなります。

## InferResponseTypeとInferRequestType

Hono Clientには、型を取り出すためのヘルパーがあります。

```ts
import type { InferRequestType, InferResponseType } from 'hono/client'

type CreateTaskRequest = InferRequestType<typeof client.tasks.$post>
type CreateTaskResponse = InferResponseType<typeof client.tasks.$post, 201>
```

たとえば、API呼び出し関数の引数に使えます。

```ts
type CreateTaskBody = InferRequestType<typeof client.tasks.$post>['json']

async function createTask(body: CreateTaskBody) {
  const res = await client.tasks.$post({
    json: body,
  })

  if (!res.ok) {
    throw new Error('Failed to create task')
  }

  return res.json()
}
```

サーバー側のスキーマを変えると、クライアント側の型も変わります。

これが、Hono RPCの大きな利点です。

## AngularからタスクAPIを利用する

Angularアプリから使う場合も、基本は同じです。

AngularのServiceにHono Clientを閉じ込めると、コンポーネントがHTTPの詳細を知らずに済みます。

コンポーネントは画面の状態やユーザー操作に集中し、APIの呼び出し方はServiceへ寄せます。
Hono Clientを使う場合も、Angularの設計としては普段のAPI Serviceと同じです。
違うのは、Serviceの内側でサーバー側のルート型を使える点です。

```ts
// task-api.service.ts
import { Injectable, inject } from '@angular/core'
import { hc } from 'hono/client'
import type { AppType } from '@server/index'

@Injectable({ providedIn: 'root' })
export class TaskApiService {
  private readonly client = hc<AppType>('http://localhost:8787')

  async listTasks(accessToken: string) {
    const res = await this.client.tasks.$get(
      {
        query: {
          limit: '20',
          offset: '0',
        },
      },
      {
        headers: {
          Authorization: `Bearer ${accessToken}`,
        },
      },
    )

    if (!res.ok) {
      throw new Error('タスク一覧の取得に失敗しました')
    }

    return res.json()
  }
}
```

コンポーネント側では、`TaskApiService` を呼ぶだけです。

```ts
const result = await this.taskApi.listTasks(accessToken)
this.tasks = result.tasks
```

Angularの `HttpClient` を使う方法もあります。

ただし、Hono Clientを使うと、サーバー側のルート型をそのまま利用できます。
APIの変更に気づきやすくなるのが利点です。

## モノレポで型を共有する

Hono RPCは、サーバーとクライアントが同じリポジトリ、または同じワークスペースにあると使いやすいです。

```txt
apps/
  api/
    src/index.ts
  web/
    src/app/task-api.service.ts
packages/
  shared/
```

クライアント側は、サーバーから型だけをimportします。

```ts
import type { AppType } from '@api/index'
```

`import type` を使うのが重要です。

これは、実行時のサーバーコードをクライアントバンドルへ含めないためです。

```ts
// 良い
import type { AppType } from '@api/index'
```

```ts
// 避けたい
import { app } from '@api/index'
```

クライアントが必要なのは型だけです。
サーバーの実体をブラウザへ持ち込む必要はありません。

## サーバーコードをクライアントへ含めない

Hono RPCでよくある落とし穴が、サーバーコードの混入です。

次のようなコードは避けます。

```ts
import app from '@api/index'
```

これは、サーバーアプリケーション本体をクライアント側へimportしています。

必要なのは `AppType` だけです。

```ts
import type { AppType } from '@api/index'
import { hc } from 'hono/client'

const client = hc<AppType>('http://localhost:8787')
```

TypeScriptの `import type` を使うことで、ビルド時に型だけが参照されます。

## 大規模アプリケーションで型を分割する

アプリケーションが大きくなると、すべてのルートをひとつの `AppType` で共有するのが重くなることがあります。

その場合は、公開したいルートだけの型をexportできます。

```ts
// src/routes/tasks.ts
export const tasksRoute = new Hono()
  .get('/', ...)
  .post('/', ...)

export type TasksRouteType = typeof tasksRoute
```

ただし、親アプリで `/tasks` に接続したあとの型が欲しい場合は、親アプリ側の `AppType` を使うほうが自然です。

```ts
const app = new Hono()
  .route('/tasks', tasksRoute)

export type AppType = typeof app
```

まずは `AppType` を1つ共有するところから始めるとよいです。

## Hono RPCの利点と制約

Hono RPCには、大きな利点があります。

| 利点 | 内容 |
| --- | --- |
| 型安全 | 入力とレスポンスを推論できる |
| 軽い | 追加のコード生成なしで使える |
| Honoらしい | ルート定義をそのまま活かせる |
| 開発体験がよい | 補完が効く |

一方で、制約もあります。

| 制約 | 内容 |
| --- | --- |
| TypeScript前提 | 型共有できる環境で強い |
| 外部公開仕様には弱い | 他言語クライアントにはOpenAPIのほうが向く |
| ルート定義の書き方に影響される | 型推論を壊さない構成が必要 |
| グローバルエラー型は注意 | `app.onError()` の型は自動で全ルートへ入らない |

Hono RPCは、TypeScript同士のサーバー・クライアントではとても強力です。

一方、外部パートナーへAPI仕様を共有したい場合や、他言語クライアントがある場合は、OpenAPIのほうが向いています。

## まとめ

この章では、Hono RPCとHono Clientを学びました。

- `AppType = typeof app` をexportする
- `hc<AppType>()` で型安全なClientを作る
- 入力型はValidatorから推論される
- レスポンス型は `c.json()` から推論される
- ルート定義はチェーンで書くと型推論を活かしやすい
- `param`、`query`、`json`、`headers` を使ってAPIを呼び出す
- AngularからもHono Clientを利用できる
- クライアント側では `import type` を使う
- TypeScript同士ならHono RPC、外部仕様共有ならOpenAPIが向いている

次章では、OpenAPIを扱います。

Hono RPCが「型を共有する」仕組みだとすれば、OpenAPIは「仕様書を共有する」仕組みです。
この2つの違いを理解すると、API設計の選択肢がぐっと広がります。
