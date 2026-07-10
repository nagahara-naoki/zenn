---
title: "JavaScriptで学ぶSingleton：共有範囲と寿命を設計する"
emoji: "🤖"
type: "tech"
topics: ["javascript", "デザインパターン", "singleton", "tanstackquery"]
published: true
---

Singletonは、ある範囲でインスタンスを1つに制御し、共有の入口を提供するパターンです。

重要なのは「世界に必ず1つ」ではなく、**どの範囲で1つなのか、いつ作り、いつ破棄するのか**を決めることです。

## moduleで共有する

ES Modulesでは、module scopeの値をexportするだけで共有できます。

```ts
// api-client.ts
class ApiClient {
  async getUser(id: number): Promise<User> {
    const response = await fetch(`/api/users/${id}`);
    return response.json();
  }
}

export const apiClient = new ApiClient();
```

同じmodule instanceをimportするコードは、同じ `apiClient` を参照します。

```ts
import { apiClient } from "./api-client";
```

JavaやC#の典型的なprivate constructor + `getInstance` をそのまま再現しなくても、module systemが生成場所を制御できます。

## 「1つ」の範囲

module singletonにも境界があります。

- ブラウザーの1ページ
- Node.jsの1process
- workerごとのrealm
- test runnerのmodule環境
- bundle内に含まれたmodule copy

SSRサーバーのmodule scopeへ利用者固有状態を置くと、複数requestで共有される危険があります。

```ts
// 危険：requestごとの情報をmodule globalへ置かない
let currentUser: User | undefined;
```

「moduleからexportしたからアプリ全体で絶対1つ」と一般化しないようにします。

## 明示的に生成を制御する

遅延生成が必要な場合は、取得関数を作れます。

```ts
let client: ApiClient | undefined;

export function getApiClient(): ApiClient {
  client ??= new ApiClient();
  return client;
}
```

ただし、この実装にはresetや破棄の仕組みがありません。connection、timer、subscriptionなどを持つresourceでは、ライフサイクルまで設計します。

多くの場合、application entry pointで1回生成し、依存として渡す方がテストしやすくなります。

```ts
const apiClient = new ApiClient();
const userService = new UserService(apiClient);
```

## TanStack QueryのQueryClient

Reactアプリでは、通常、コンポーネントのライフサイクル中に安定した `QueryClient` を使います。

```tsx
function App() {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 30_000,
          },
        },
      }),
  );

  return (
    <QueryClientProvider client={queryClient}>
      <Routes />
    </QueryClientProvider>
  );
}
```

renderごとに `new QueryClient()` するとcacheが毎回作り直されます。安定したinstanceをproviderへ渡す必要があります。

この点は「アプリのclient側ライフサイクルで1つの共有インスタンスを使う」というSingleton的な考え方です。

ただしQueryClientのAPIがGoF Singletonを強制しているわけではありません。独立したcacheが必要なら複数作れます。

## SSRではrequestごとに考える

サーバー上のmodule scopeで1つのQueryClientを全requestへ共有すると、cache dataが利用者間で混ざる危険があります。

```ts
function createRequestQueryClient(): QueryClient {
  return new QueryClient();
}
```

SSRでは、requestごとにclientを作り、dehydrate後に適切に破棄する構成を検討します。

つまりscopeは次のように変わります。

| 環境 | 典型的な共有範囲 |
| --- | --- |
| ブラウザー | アプリのライフサイクル |
| SSR | request |
| テスト | test caseまたはtest suite |

## Jotaiのdefault store

Jotaiのprovider-less modeでは、`getDefaultStore` がdefault storeを返します。

```ts
const store = getDefaultStore();
```

同じdefault storeを共有する点はSingleton的です。

一方、Providerや `createStore` を使えば、subtree・request・testごとにstoreを分離できます。共有が便利か、分離が必要かで選びます。

## Singletonが向くもの

- client側アプリで共有するcache client
- statelessなregistry
- process単位の設定読取
- 重いが共有可能なservice

## 向かないもの

- 利用者・requestごとの状態
- testで独立させたい変更可能状態
- 複数設定を同時に使うclient
- 明示的な破棄が必要なのに寿命を管理できないresource

## テストへの影響

Singletonの状態はtest間で残りやすく、実行順序依存を生みます。

```ts
beforeEach(() => {
  queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
    },
  });
});
```

テストでは新しいinstanceを作れるAPIを残し、productionだけ共有する構成が扱いやすくなります。

## 判断基準

Singletonにする前に確認します。

1. 共有すべきscopeはpage、request、processのどれか
2. 状態は利用者間で共有して安全か
3. 並列テストを分離できるか
4. reset・disposeが必要か
5. 将来、複数設定を同時に使う可能性はないか

## まとめ

Singletonで最も重要なのは、instance数よりscopeとlifecycleです。

- ES Modulesで自然に共有できる
- module singletonにもrealmやbundleの境界がある
- clientで1つでも、SSRではrequestごとが適切な場合がある
- 変更可能なglobal stateには慎重になる
- テスト用に新しいinstanceを作れる設計を残す

## 参考資料

- [TanStack Query: Stable Query Client](https://tanstack.com/query/v5/docs/eslint/stable-query-client)
- [TanStack Query: Advanced Server Rendering](https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr)
- [Jotai Docs: Store](https://jotai.org/docs/core/store)
- [Jotai Docs: Next.js](https://jotai.org/docs/guides/nextjs)
