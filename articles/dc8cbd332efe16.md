---
title: "JavaScriptで学ぶSingleton：共有範囲とライフサイクルを設計する"
emoji: "🤖"
type: "tech"
topics: ["javascript", "typescript", "デザインパターン", "singleton", "tanstackquery"]
published: true
---

設定、cache、connection pool、監視registryなど、同じ範囲で1つのinstanceを共有したいresourceがあります。そこで「どこからでも使えるglobal object」を作ると手軽ですが、共有範囲が曖昧なままでは、利用者間のdata混在、testの順序依存、破棄できない接続を生みます。

Singleton（シングルトン）は、あるclassのinstanceを1つに制御し、そのinstanceへのglobalなaccess pointを提供する生成パターンです。しかし現代のJavaScriptで効くのは、古典的な `private constructor + getInstance()` の形より、「どの範囲で1つか」「誰が作り、いつ破棄するか」です。

本記事ではES Modulesによる共有、遅延初期化、非同期生成、SSR、testを段階的に検討します。TanStack QueryのQueryClientとJotaiのdefault storeも公式資料に照らして読みますが、公式のGoF分類と教育的な類推は区別します。

## 無制限なglobal stateが起こす問題

login中のユーザーをmodule変数へ置く例です。

```ts
type User = Readonly<{ id: string; name: string }>;

let currentUser: User | undefined;

export function setCurrentUser(user: User): void {
  currentUser = user;
}

export function getCurrentUser(): User | undefined {
  return currentUser;
}
```

browserの単一tabだけなら動いても、SSR serverでは複数requestが同じmodule instanceを共有し得ます。request Aが設定したユーザーを、同時実行のrequest Bが読むdata leakにつながります。testでも前のcaseが値を残すと、単独実行では通るのにsuiteでは落ちます。

問題の原因はinstance数そのものより、共有してはいけない状態を広すぎるscopeへ置いたことです。ユーザー、locale、request ID、transactionなどはrequest固有です。Singletonを検討する前に、その値を共有して安全かを問います。

```ts
// request固有値は明示的なcontextで渡す
type RequestContext = Readonly<{
  user: User | undefined;
  requestId: string;
}>;

async function handleRequest(context: RequestContext): Promise<Response> {
  return Response.json({ userId: context.user?.id ?? null });
}
```

## Singletonを正確に定義する

GoFのSingletonは、classがinstanceを1つだけ持つことを保証し、そのinstanceへアクセスするpointを提供します。生成数の制御とglobal accessが組み合わさったパターンです。古典的なobject-oriented言語では、private constructor、static field、static `getInstance` がよく使われます。

ただし「1つ」は宇宙全体で1つという意味ではありません。JavaScriptではbrowser tab、Realm、Worker、Node.js process、module loader、package copy、test sandboxなどの境界があります。同じソースがbundleへ複数回含まれれば、それぞれが別のmodule状態を持つ場合があります。processを複数起動すればmemoryは共有されません。

また、1つに制限することと、どこからでも直接importできることは別に考えられます。composition rootで1instanceを作って依存として渡せば、利用数は1つでもglobal accessは避けられます。多くのアプリではこちらの方が依存関係とtestを明示できます。

## ES Modulesで共有する最小形

ES Modulesでは、同じmodule instanceのexportを複数箇所からimportすると、同じobject参照を利用できます。単純なclient側アプリでは、古典的なclass Singletonより小さく書けます。

```ts
// api-client.ts
class ApiClient {
  constructor(private readonly baseUrl: string) {}

  async getUser(id: string): Promise<User> {
    const response = await fetch(
      new URL(`/users/${encodeURIComponent(id)}`, this.baseUrl),
    );
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return (await response.json()) as User;
  }
}

export const apiClient = new ApiClient("https://api.example.com");
```

```ts
import { apiClient } from "./api-client.js";

const user = await apiClient.getUser("u_1");
```

この形はeagerに生成され、import時の副作用を持ちます。設定値をtestごとに変える、初期化失敗を扱う、connectionを破棄する、といった要件には弱い設計です。またmodule cacheの単位は実行環境とloaderに依存するため、「ESMだから絶対に全体で1つ」と説明してはいけません。

## composition rootで1つだけ作る

多くの場合、Singleton classを強制せず、application entry pointで1回生成して依存へ渡す方が明快です。

```ts
interface UserReader {
  getUser(id: string): Promise<User>;
}

class UserService {
  constructor(private readonly users: UserReader) {}

  async displayName(id: string): Promise<string> {
    return (await this.users.getUser(id)).name;
  }
}

const client = new ApiClient("https://api.example.com");
const userService = new UserService(client);
```

実行中の `ApiClient` は1つでも、`UserService` はglobalな取得関数を呼びません。依存がconstructorへ現れ、testではfakeを渡せます。将来、tenantごとに別clientを使う場合もclassの制約を壊す必要がありません。

```ts
const fakeUsers: UserReader = {
  async getUser(id) {
    return { id, name: "Test User" };
  },
};

const testService = new UserService(fakeUsers);
```

## 遅延初期化を安全にする

使われない可能性がある高価なresourceは、最初の利用時に生成したくなります。同期生成ならmodule内の変数と取得関数で表せます。

```ts
class MetricsRegistry {
  private readonly counters = new Map<string, number>();

  increment(name: string): void {
    this.counters.set(name, (this.counters.get(name) ?? 0) + 1);
  }
}

let metrics: MetricsRegistry | undefined;

export function getMetrics(): MetricsRegistry {
  metrics ??= new MetricsRegistry();
  return metrics;
}
```

同期実行では途中に `await` がないため同じevent loop上で二重生成されません。ただし別Worker、別process、同じpackageの複製までは防げず、取得関数の乱用は依存も隠します。

非同期生成では、完成instanceだけをcacheすると競合します。2つの呼び出しが最初の `await` 前に「未生成」と判断し、接続を2つ作る可能性があります。完成値ではなく初期化Promiseを直ちにcacheします。

```ts
type Database = {
  query(sql: string): Promise<unknown>;
  close(): Promise<void>;
};

type CreateDatabase = () => Promise<Database>;

let databasePromise: Promise<Database> | undefined;

export function getDatabase(createDatabase: CreateDatabase): Promise<Database> {
  databasePromise ??= createDatabase();
  return databasePromise;
}
```

初期化が失敗したPromiseを永遠にcacheすると、設定を直した後も再試行できません。逆に毎回自動再試行すると障害時に接続stormを起こします。次の例は失敗時にcacheを消しますが、backoffや最大回数は呼び出し側または専用policyで制御します。

```ts
export function getRetryableDatabase(
  createDatabase: CreateDatabase,
): Promise<Database> {
  if (!databasePromise) {
    databasePromise = createDatabase().catch((error) => {
      databasePromise = undefined;
      throw error;
    });
  }
  return databasePromise;
}
```

## 破棄まで含めてライフサイクルを設計する

connection、timer、subscription、file handleを持つSingletonは、作成だけでなく終了処理が必要です。`close` の所有者を決め、process終了、test終了、hot reloadなど適切な時点で呼びます。

```ts
export async function closeDatabase(): Promise<void> {
  const pending = databasePromise;
  databasePromise = undefined;
  if (!pending) return;

  const database = await pending;
  await database.close();
}
```

closeとqueryが同時に走る場合や、close後の再生成を許すかも契約です。moduleへ本番からも呼べる `resetForTest` を足すより、resource ownerとなるcontainerをtestごとに生成・破棄する方が安全です。

```ts
class AppResources {
  readonly database: Promise<Database>;

  constructor(createDatabase: CreateDatabase) {
    this.database = createDatabase();
  }

  async close(): Promise<void> {
    await (await this.database).close();
  }
}

async function main(
  createDatabase: CreateDatabase,
  runApplication: (resources: AppResources) => Promise<void>,
): Promise<void> {
  const resources = new AppResources(createDatabase);
  try {
    await runApplication(resources);
  } finally {
    await resources.close();
  }
}
```

このcontainerは複数作れますが、entry pointが1つだけ作るためapplication scopeでは1つです。所有権と破棄を明示でき、test suiteごとの隔離もしやすくなります。

## SSRではrequest scopeを分ける

client側の1applicationで安全な共有状態も、長寿命serverでは利用者間に広がります。SSRでrequest固有dataを含むcache client、store、locale providerをmodule singletonにしてはいけません。

```ts
import { QueryClient } from "@tanstack/react-query";

type RequestServices = Readonly<{
  queryClient: QueryClient;
  user: User | undefined;
}>;

function createRequestServices(user: User | undefined): RequestServices {
  return {
    queryClient: new QueryClient(),
    user,
  };
}

async function renderRequest(
  user: User | undefined,
  renderApp: (services: RequestServices) => Promise<string>,
): Promise<string> {
  const services = createRequestServices(user);
  return renderApp(services);
}
```

「requestごとに1つ」はscoped singletonと呼ばれることがありますが、GoFのglobal access pointとは異なります。用語だけで判断せず、instanceの所有者と共有範囲を明示します。

```text
Node.js process
  ├─ request A ─ QueryClient A / User A
  └─ request B ─ QueryClient B / User B
```

process共通にする候補は不変な設定、metrics registry、connection poolなどです。複数processを越えた一意性が必要な採番やlockは、DBや分散coordinationへ任せます。

## testの順序依存を防ぐ

共有instanceの可変状態はtest間に残り、実行順や並列化で壊れます。production singletonを直接使わず、生成可能なFactoryと小さなinterfaceを用意します。

```ts
import assert from "node:assert/strict";
import test from "node:test";

test("UserService can be tested without the shared client", async () => {
  const users: UserReader = {
    async getUser(id) {
      return { id, name: "Ada" };
    },
  };

  const service = new UserService(users);
  assert.equal(await service.displayName("u_1"), "Ada");
});
```

cache clientもcaseごとに新instanceを作ります。global instanceを `beforeEach` でclearすると並列testが競合します。

```ts
import { QueryClient } from "@tanstack/react-query";

function createTestQueryClient(): QueryClient {
  return new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  });
}

test("isolated cache", async () => {
  const queryClient = createTestQueryClient();
  try {
    queryClient.setQueryData(["user", "u_1"], { name: "Ada" });
    assert.deepEqual(queryClient.getQueryData(["user", "u_1"]), { name: "Ada" });
  } finally {
    queryClient.clear();
  }
});
```

## TanStack QueryのQueryClientをSingleton的に読む

TanStack Queryの公式quick startは `new QueryClient()` でclientを作り、`QueryClientProvider` へ渡す例を示します。QueryClientはquery cacheやmutation cacheと結び付き、default optionも保持します。React componentのrenderごとに新しくするとcacheが失われるため、client applicationのlifecycleでは安定した同一instanceを使います。

```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState } from "react";

function App() {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: { queries: { staleTime: 30_000 } },
      }),
  );

  return (
    <QueryClientProvider client={queryClient}>
      <main>Application</main>
    </QueryClientProvider>
  );
}
```

TanStack QueryのESLintルール `stable-query-client` も、QueryClientはapplication lifecycleで1つの安定instanceを持つべきと説明しつつ、serverではasync server componentが1requestに1回実行されるため例外を示しています。高度なSSR公式guideでは、requestごとにQueryClientを作る構成が案内されています。clientでの「1つ」をserverへそのまま広げない好例です。

ただしQueryClientのconstructorはpublicで、複数instanceも作れます。TanStack Query公式がGoF Singletonとして分類しているわけではありません。本記事では「定めたlifecycle内で安定した1instanceを共有する」というSingleton的な運用を学ぶ教育的な類推として扱っています。instance数を強制しているのではなく、Providerがscopeを決めています。

## Jotaiのdefault storeをSingleton的に読む

Jotai公式のStore APIでは、`createStore()` が新しい空storeを作り、`getDefaultStore()` がprovider-less modeで使われるdefault storeを返すと説明されています。

```ts
import { createStore, getDefaultStore } from "jotai/vanilla";

const sharedStore = getDefaultStore();
const isolatedStore = createStore();
```

default storeを共有する点はmodule Singletonに近い一方、`createStore` やReactのProviderを使えばsubtree、test、requestごとに別storeを作れます。Jotai公式はこれをGoF Singletonとして分類していません。共有が便利なprovider-less modeと、隔離可能な明示storeを両方提供している実在設計として読むのが正確です。

SSRでdefault storeへ利用者固有状態を置く場合は共有範囲を確認します。公式のNext.js guideも、server side renderingではrequestごとにstoreを保持する必要性を説明しています。libraryがdefault instanceを提供していても、すべての環境でそれを使うべきという意味ではありません。

## より単純な代替案とトレードオフ

不変な定数なら、object instanceを作らず通常の `export const` で十分です。stateを持たない純粋関数も共有instanceを必要としません。依存を1か所で組み立てたいだけならcomposition root、requestごとの値なら引数やcontext、階層ごとの状態ならProviderを使えます。

Singletonは生成costを抑え、cacheやconnection poolを共有できます。一方、依存をimportの裏へ隠し、初期化順序とtest隔離を難しくします。lazy初期化でも設定タイミングが見えにくくなる場合があります。

## 失敗しやすい設計

- 「アプリで1つ」を、Realm、Worker、processを越えて1つと誤解する。
- SSRのmodule scopeへユーザーやrequest固有cacheを置き、利用者間で共有する。
- 非同期初期化の完成値だけをcacheし、同時呼び出しで二重生成する。
- 失敗したPromiseを永遠にcacheする、または無制限に再接続する。
- connectionやtimerを作るだけで、closeするownerと時点を決めない。
- global accessorをあらゆるclassから呼び、依存関係を隠す。
- testが共有instanceをclearし、並列testと競合する。
- packageの複製やhot reload下でもinstanceが必ず1つだと仮定する。

Singletonへ可変設定を持たせ、途中で書き換える設計にも注意します。あるrequestの途中で設定が変われば、処理前半と後半で異なる規則を使います。設定snapshotを不変objectとして生成し、変更は新しいscopeや明示的なreload手順で行う方が予測しやすくなります。

## 使うかどうかの判断基準

Singleton的な共有が向くのは、同じscope内で本当に1つの状態を共有する必要があり、複数生成がresource競合や一貫性低下を起こし、lifecycleを明確に所有できる場合です。client applicationのcache client、process単位のmetrics registry、connection poolなどが候補です。

導入前には「1つとはtab、request、processのどれか」「共有する状態に利用者固有dataがないか」「複数processでも整合するか」「いつ初期化し、失敗時にどう再試行し、誰が破棄するか」「testごとに隔離できるか」「global accessを避けて依存注入できないか」を確認します。

instanceを制限することが目的になっている、将来複数設定を使う可能性が高い、状態を独立testしたい、request固有である場合は避けます。必要なのが単なる共有よりscope管理なら、ProviderやDI containerのlifetime、明示的なresource containerの方が意図を表せます。

## まとめ

Singletonで最初に見るのは、static fieldの書き方ではありません。scopeとlifecycleです。ES Modulesは同一module instance内の共有を簡潔にしますが、Worker、process、bundle、SSR requestを越えた一意性までは保証しません。まずcomposition rootで1instanceを作って渡す方法を検討し、global accessが本当に必要な場合だけ取得APIを置きます。

非同期生成ではPromiseをcacheし、初期化失敗と再試行を設計します。resourceにはcloseするownerを定め、request固有dataはrequest scopeへ隔離します。TanStack QueryとJotaiは安定した共有instanceと明示的な分離の両方を学べる例ですが、公式のGoF分類ではありません。実行環境の境界を踏まえて「どこで1つか」を説明できることが、安全な共有設計の出発点です。

## 参考資料

- Erich Gammaほか『Design Patterns: Elements of Reusable Object-Oriented Software』Singleton章（Addison-Wesley, 1994）
- [ECMAScript仕様：Modules](https://tc39.es/ecma262/multipage/ecmascript-language-scripts-and-modules.html#sec-modules)
- [TanStack Query公式：Quick Start](https://tanstack.com/query/latest/docs/framework/react/quick-start)
- [TanStack Query公式：Stable Query Client](https://tanstack.com/query/latest/docs/eslint/stable-query-client)
- [TanStack Query公式：Advanced Server Rendering](https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr)
- [TanStack Query公式ソース：QueryClient](https://github.com/TanStack/query/blob/main/packages/query-core/src/queryClient.ts)
- [Jotai公式：Store](https://jotai.org/docs/core/store)
- [Jotai公式：Next.js](https://jotai.org/docs/guides/nextjs)
- [Jotai公式ソース：store.ts](https://github.com/pmndrs/jotai/blob/main/src/vanilla/store.ts)
