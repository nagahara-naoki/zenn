---
title: "JavaScriptで学ぶSingletonパターン！TanStack QueryのQueryClientから見る「1つだけ共有する設計」"
emoji: "🤖"
type: "tech"
topics: ["javascript", "デザインパターン"]
published: true
---
アプリ全体で、同じインスタンスを使い回したいことはありませんか？

例えば、データ取得用のクライアントです。

```js
const clientA = createApiClient();
const clientB = createApiClient();
const clientC = createApiClient();
```

これでも動くかもしれません。

でも、もしこのclientが内部にキャッシュを持っていたらどうでしょうか？

```js
function createApiClient() {
  const cache = new Map();

  return {
    async get(path) {
      if (cache.has(path)) {
        return cache.get(path);
      }

      const response = await fetch(path);
      const data = await response.json();

      cache.set(path, data);

      return data;
    },
  };
}
```

このclientを何度も作ると、cacheも別々になります。

```js
const clientA = createApiClient();
const clientB = createApiClient();
```

`clientA` が取得したデータを、`clientB` は知りません。

本当はアプリ全体で同じcacheを使いたいのに、インスタンスを作り直すことで状態が分かれてしまいます。

こういうときに出てくるのが **Singletonパターン** です。

---

## Singletonパターンとは？

Singletonパターンは、**あるインスタンスを1つだけ作り、それを共有するパターン**です。

簡単に言うと、

> 何度使っても、同じインスタンスを参照するようにする

という考え方です。

例えば、アプリ全体で1つのAPI clientを共有します。

```js
const apiClient = createApiClient();

export { apiClient };
```

他のファイルでは、これをimportして使います。

```js
import { apiClient } from "./apiClient.js";

apiClient.get("/users");
```

新しくclientを作り直すのではなく、同じclientを共有します。

これがSingleton的な使い方です。

---

## Singletonを使わない場合の問題

次のように、コンポーネントや関数の中で毎回clientを作る例を考えます。

```js
function getUser(id) {
  const client = createApiClient();

  return client.get(`/users/${id}`);
}
```

このコードには、次の問題があります。

1. 呼ぶたびに新しいclientが作られる
2. client内部のcacheや設定が共有されない
3. 重いインスタンスなら生成コストが無駄になる

特に、次のようなものは何度も作ると問題になりやすいです。

- キャッシュを持つclient
- 状態管理store
- DB接続client
- WebSocket接続
- ロガー
- 設定オブジェクト

もちろん、毎回新しく作った方がよいものもあります。

Singletonは「何でも1つにする」ためのパターンではありません。

**1つにした方が自然なものを、適切なスコープで共有する** ための考え方です。

---

## Singletonで改善する

まずは、シンプルなSingletonを書いてみます。

```js
let apiClient = null;

function getApiClient() {
  if (apiClient) {
    return apiClient;
  }

  apiClient = createApiClient();

  return apiClient;
}
```

使う側です。

```js
const clientA = getApiClient();
const clientB = getApiClient();

console.log(clientA === clientB); // true
```

`getApiClient()` を何度呼んでも、同じインスタンスが返ります。

---

## ES Modulesならもっと自然に書ける

JavaScriptでは、クラスでSingletonを頑張って作るより、ES Modulesで共有する方が自然なことが多いです。

```js
// apiClient.js
function createApiClient() {
  const cache = new Map();

  return {
    async get(path) {
      if (cache.has(path)) {
        return cache.get(path);
      }

      const response = await fetch(path);
      const data = await response.json();

      cache.set(path, data);

      return data;
    },
  };
}

export const apiClient = createApiClient();
```

使う側です。

```js
import { apiClient } from "./apiClient.js";

const user = await apiClient.get("/users/1");
```

このように、モジュールのトップレベルで1回作ってexportすれば、同じインスタンスを共有できます。

JavaScriptでは、この形がかなり実務的です。

---

## Singletonで何が良くなるのか？

### 1. 同じ状態やcacheを共有できる

clientがcacheを持つ場合、同じclientを共有すればcacheも共有できます。

```js
export const apiClient = createApiClient();
```

別々の場所から使っても、同じインスタンスを参照できます。

---

### 2. 生成コストを減らせる

重いインスタンスを毎回作らずに済みます。

```js
export const logger = createLogger();
export const queryClient = new QueryClient();
```

作成コストが高いものほど、共有する価値があります。

---

### 3. 設定を統一できる

API clientやloggerの設定を1か所にまとめられます。

```js
export const apiClient = createApiClient({
  baseUrl: "https://api.example.com",
  timeout: 5000,
});
```

あちこちでバラバラに設定するより安全です。

---

## 有名OSSではどう使われている？

Singletonパターンに近い考え方は、**TanStack Queryの `QueryClient`** に見ることができます。

TanStack Queryでは、Reactアプリに `QueryClientProvider` を置き、`QueryClient` を渡します。

```tsx
import {
  QueryClient,
  QueryClientProvider,
} from "@tanstack/react-query";

const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Home />
    </QueryClientProvider>
  );
}
```

`QueryClient` は、TanStack Queryのcache管理の中心になるインスタンスです。

そのため、レンダーのたびに新しく作ると問題になります。

---

## QueryClientをSingleton的に見る

TanStack Queryの公式ESLintルールでは、`QueryClient` は `QueryCache` を含むため、アプリケーションのライフサイクル中は1つの `QueryClient` インスタンスを作るべきで、レンダーごとに新しく作るべきではないと説明されています。

これはSingleton的な考え方です。

悪い例です。

```tsx
function App() {
  const queryClient = new QueryClient();

  return (
    <QueryClientProvider client={queryClient}>
      <Home />
    </QueryClientProvider>
  );
}
```

`App` が再レンダーされるたびに、新しい `QueryClient` が作られる可能性があります。

すると、cacheも分かれてしまいます。

良い例です。

```tsx
const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Home />
    </QueryClientProvider>
  );
}
```

または、React stateで初回だけ作る方法もあります。

```tsx
function App() {
  const [queryClient] = useState(() => new QueryClient());

  return (
    <QueryClientProvider client={queryClient}>
      <Home />
    </QueryClientProvider>
  );
}
```

どちらも、アプリのライフサイクル中に安定した `QueryClient` を使うための書き方です。

---

## QueryClientが1つである意味

TanStack Queryの `QueryCache` は、queryのdata、meta information、stateを保存するstorage mechanismとして説明されています。

つまり、cacheの中心です。

`QueryClient` は、そのcacheとやり取りするための入口になります。

そのため、`QueryClient` を何個も作ると、cacheも分かれます。

```txt
QueryClient A
  └ QueryCache A

QueryClient B
  └ QueryCache B
```

同じ `["todos"]` というquery keyを使っていても、別々の `QueryClient` なら別々のcacheになります。

だから、通常のReactアプリでは、1つの安定した `QueryClient` をProviderに渡します。

```tsx
<QueryClientProvider client={queryClient}>
  <App />
</QueryClientProvider>
```

これは、SingletonパターンをReactのProviderスコープで扱っている例として見ると分かりやすいです。

---

## ただし「グローバルに1つ」が常に正解ではない

ここがSingletonパターンで一番大事です。

Singletonは便利ですが、何でもグローバルに1つにすればよいわけではありません。

特に注意したいのは、SSRやテストです。

例えば、サーバー側でユーザーごとの状態をグローバルに持つと危険です。

```js
const globalState = {
  currentUser: null,
};

function handleRequest(user) {
  globalState.currentUser = user;

  return renderPage();
}
```

複数ユーザーのリクエストが同じプロセスで処理されると、状態が混ざる可能性があります。

この場合は、リクエストごとに状態を作るべきです。

```js
function handleRequest(user) {
  const requestState = {
    currentUser: user,
  };

  return renderPage(requestState);
}
```

Singletonで大事なのは、**どの範囲で1つにするのか** です。

---

## 補足：JotaiのgetDefaultStoreもSingleton的

補足として、Jotaiの `getDefaultStore` もSingleton的に見ることができます。

Jotaiには、Providerなしでatomを使うprovider-less modeがあります。

```tsx
const countAtom = atom(0);

function Counter() {
  const [count, setCount] = useAtom(countAtom);

  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
}
```

このとき、Jotaiは暗黙的なdefault storeを使います。

```ts
import { getDefaultStore } from "jotai";

const store = getDefaultStore();
```

Jotai公式ドキュメントでは、`getDefaultStore` はprovider-less modeで使われるdefault storeを返すAPIとして説明されています。

これは、共有storeを返すという意味でSingleton的です。

---

## JotaiのSSR注意点

JotaiのNext.jsガイドでは、Providerを使わない場合、暗黙的なglobal storeが使われると説明されています。

そしてSSRでは、このglobal storeがリクエスト間で残ることで、次のような問題が起きる可能性があると説明されています。

- memory leak
- data leak
- stale data

そのため、SSRでは `Provider` を使い、storeをrequestやrenderのスコープに分けることが推奨されています。

```tsx
import { Provider } from "jotai";

function App({ Component, pageProps }) {
  return (
    <Provider>
      <Component {...pageProps} />
    </Provider>
  );
}
```

ここから学べるのは、Singletonは便利だけれど、スコープを間違えると危険だということです。

---

## TanStack QueryとJotaiから学べること

TanStack QueryとJotaiの例を見ると、Singletonで大事なのは「1つにすること」そのものではありません。

大事なのは、**どのスコープで1つにするか** です。

| 対象                 | よくあるスコープ              |
| -------------------- | ----------------------------- |
| QueryClient          | アプリのライフサイクル中に1つ |
| Jotai default store  | provider-less modeで共有      |
| Jotai Provider store | Providerごとに1つ             |
| SSRのユーザー状態    | リクエストごとに1つ           |
| テスト用store        | テストケースごとに1つ         |

このように、同じ「1つ」でも意味が違います。

Singletonを使うときは、必ずスコープを考える必要があります。

---

## 実務でSingletonを使うならどこ？

Singletonパターンは、次のような場面で使いやすいです。

- QueryClientのようなcache管理インスタンス
- APIクライアント
- ロガー
- アプリ設定
- WebSocket接続
- DBクライアント
- クライアントサイドの状態管理store

例えば、loggerです。

```js
function createLogger() {
  return {
    info(message) {
      console.log(`[INFO] ${message}`);
    },

    error(message) {
      console.error(`[ERROR] ${message}`);
    },
  };
}

export const logger = createLogger();
```

使う側です。

```js
import { logger } from "./logger.js";

logger.info("アプリを起動しました");
```

loggerの設定や出力形式を1か所にまとめられます。

---

## 使いすぎには注意

Singletonは便利ですが、かなり注意が必要です。

特に、状態を持つSingletonは危険になりやすいです。

```js
export const authState = {
  currentUser: null,
};
```

このような状態がどこからでも変更できると、処理の流れが追いづらくなります。

また、テストでも問題が起きやすいです。

```js
test("A", () => {
  authState.currentUser = { name: "Taro" };
});

test("B", () => {
  // 前のテストの状態が残っているかもしれない
});
```

テストごとに状態を分けたいなら、SingletonよりFactoryの方が向いていることがあります。

```js
function createAuthState() {
  let currentUser = null;

  return {
    login(user) {
      currentUser = user;
    },

    getCurrentUser() {
      return currentUser;
    },
  };
}
```

テストでは毎回新しい状態を作れます。

```js
const authState = createAuthState();
```

Singletonを使う前に、本当に共有してよい状態かを確認しましょう。

---

## Singletonにしてよいかの判断

Singletonにしてよい可能性が高いものです。

- 生成コストが高い
- cacheや接続を共有したい
- アプリ全体で設定を統一したい
- 複数作ると不整合が起きる
- ライフサイクル中に安定していてほしい

逆に、Singletonにしない方がよいものです。

- ユーザーごとの状態
- リクエストごとの状態
- テストごとに分けたい状態
- tenantごとに異なる設定
- どこからでも変更できる可変状態

大事なのは、「1つにできるか」ではなく、**1つにして安全か** です。

---

## まとめ

Singletonパターンは、**あるインスタンスを1つだけ作り、それを共有するパターン**です。

JavaScriptでは、ES Modulesでインスタンスを1回作ってexportする形がよく使われます。

TanStack Queryの `QueryClient` は、`QueryCache` を持つため、通常のReactアプリではアプリのライフサイクル中に1つの安定したインスタンスを使うのが基本です。

Jotaiの `getDefaultStore` も、provider-less modeで使われるdefault storeを返すAPIとしてSingleton的に見ることができます。

ただし、Singletonはグローバル状態になりやすく、SSRやテストでは危険になることがあります。

Singletonで大事なのは、単に1つだけ作ることではありません。

大事なのは、**どのスコープで1つにすべきかを決めること** です。

インスタンスを共有したくなったら、こう考えてみるとよいです。

> このインスタンスは、アプリ全体で1つにして安全？  
> それとも、リクエスト・Provider・テストごとに分けるべき？

この問いに答えられるなら、Singletonパターンはかなり実務で役立ちます。

---

## 参考リンク

- [TanStack Query Stable Query Client](https://tanstack.com/query/v5/docs/eslint/stable-query-client)
- [TanStack Query QueryCache](https://tanstack.com/query/latest/docs/reference/QueryCache)
- [TanStack Query Important Defaults](https://tanstack.com/query/v4/docs/framework/react/guides/important-defaults)
- [Jotai Store docs](https://jotai.org/docs/core/store)
- [Jotai Next.js guide](https://www.mintlify.com/pmndrs/jotai/guides/nextjs)
