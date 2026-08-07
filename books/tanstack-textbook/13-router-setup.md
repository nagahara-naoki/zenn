---
title: "TanStack Routerの導入"
---

ここから、状態の2つめの置き場所に移ります。URLです。

サーバー状態はTanStack Queryに預けました。次は、画面遷移と、URLに載せたい状態を扱います。担当するのはTanStack Routerです。

この章では、Routerを導入してファイルからルートを組み立てるところまで進めます。URLに状態を持たせる話は、その先の章で扱います。

## なぜルーティングに型安全を求めるのか

Reactのルーティングで、こんなコードを書いたことがあるはずです。

```tsx
<a href="/tasks/1">タスク詳細</a>
navigate(`/taks/${id}`); // 綴りを間違えている
```

2行目は`tasks`が`taks`になっています。それでもビルドは通り、テストで踏まなければ本番まで到達します。踏んだユーザーには「404 Not Found」が出るだけです。

URLに関わる情報は、たいてい文字列として扱われます。パス、パスの中の変数、検索条件。どれも書き間違えられるのに、コンパイラは何も言いません。

```tsx
// パラメータ名を間違えた
const { taskID } = useParams(); // 正しくは taskId → undefined になる

// 検索条件が数値のつもりで文字列だった
const page = searchParams.get('page'); // string | null
```

TanStack Routerは、この領域に型を持ち込みます。定義したルートの一覧が型になり、存在しないパスへのリンクはコンパイルエラーになります。パスの変数も検索条件も、型のついた値として受け取れます。

## インストールと設定

必要なパッケージは3つです。

```sh
npm i @tanstack/react-router @tanstack/react-router-devtools
npm i -D @tanstack/router-plugin
```

`@tanstack/router-plugin`は、ファイルからルートの定義を生成するViteプラグインです。設定に追加します。

```ts:vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { tanstackRouter } from '@tanstack/router-plugin/vite';

export default defineConfig({
  plugins: [
    // Routerのプラグインは react() より前に置く
    tanstackRouter({ target: 'react', autoCodeSplitting: true }),
    react(),
  ],
});
```

順番に意味があります。Routerのプラグインを`react()`より前に置いてください。逆にすると、生成が間に合わずにビルドが失敗します。

`autoCodeSplitting: true`は、ルートごとにコードを分割する設定です。タスク一覧の画面を開いたとき、設定画面のコードは読み込まれません。有効にしておいて損はありません。

## File-based Routingという考え方

TanStack Routerには2つの書き方があります。ファイルの配置からルートを決めるFile-based Routingと、コードでルートツリーを組み立てるCode-based Routingです。

本書ではFile-based Routingを使います。ファイルを置くだけでルートが増え、URLとファイルの対応が目で追えるからです。

```mermaid
flowchart LR
  F["src/routes/ に<br/>ファイルを置く"] --> P["router-plugin が検知"]
  P --> G["routeTree.gen.ts<br/>を自動生成"]
  G --> T["ルートの一覧が<br/>型として使える"]
```

生成される`src/routeTree.gen.ts`は、機械が書くファイルです。冒頭にも「変更しないでください」と書かれています。Gitに含めるかどうかは好みが分かれますが、含めておくと、環境を作り直した直後でも型が効いた状態から始められます。

### ファイル名の規則

規則を覚えれば、URLの設計がそのままディレクトリの設計になります。

| ファイル | できるURL |
|---|---|
| `routes/__root.tsx` | すべてのルートの親（URLは持たない） |
| `routes/index.tsx` | `/` |
| `routes/about.tsx` | `/about` |
| `routes/tasks/index.tsx` | `/tasks` |
| `routes/tasks/$taskId.tsx` | `/tasks/123`（変数） |
| `routes/tasks/route.tsx` | `/tasks`配下の共通レイアウト |
| `routes/reports.monthly.tsx` | `/reports/monthly`（ドット区切り） |
| `routes/_settings.tsx` | URLに現れないレイアウト |
| `routes/_settings/profile.tsx` | `/profile`（`_settings`の内側） |
| `routes/files/$.tsx` | `/files/`以下のすべて |
| `routes/-shared/Helper.tsx` | ルートにならない（除外される） |

3つ、注意したい規則があります。

1つめは、`_`で始まる名前がURLに現れないことです。`_settings/profile.tsx`のURLは`/settings/profile`ではなく`/profile`になります。「認証が必要な画面をまとめたい。でもURLに`/auth`とは出したくない」という要求に応えるための仕組みです。

2つめは、`-`で始まるディレクトリがルートとして扱われないことです。ルートの近くに置きたいコンポーネントや関数の置き場所になります。

3つめは、ドット区切りとディレクトリが同じ結果になることです。`reports.monthly.tsx`と`reports/monthly.tsx`は同じURLです。階層が浅いうちはドット区切りのほうが見通しがよく、深くなるとディレクトリのほうが扱いやすくなります。

本書のアプリは、この規則にしたがって次の形になります。

```text
src/routes/
├── __root.tsx          … 全ページ共通のレイアウト
├── index.tsx           … /
└── tasks/
    ├── index.tsx       … /tasks
    └── $taskId.tsx     … /tasks/123
```

ディレクトリを見ればURLがわかり、URLを見ればファイルの場所がわかります。この対応が崩れないことが、File-based Routingの効きどころです。

## 最初のルートを作る

すべての親になる`__root.tsx`から書きます。

```tsx:src/routes/__root.tsx
import { createRootRoute, Link, Outlet } from '@tanstack/react-router';
import { TanStackRouterDevtools } from '@tanstack/react-router-devtools';

export const Route = createRootRoute({
  component: RootLayout,
  notFoundComponent: () => <p>ページが見つかりません</p>,
});

function RootLayout() {
  return (
    <>
      <header>
        <nav>
          <Link to="/">ホーム</Link> <Link to="/tasks">タスク</Link>
        </nav>
      </header>

      <main>
        <Outlet />
      </main>

      <TanStackRouterDevtools />
    </>
  );
}
```

`Route`という名前で`export`するのが約束です。プラグインはこの名前を探します。

`<Outlet />`が、子のルートが描画される穴です。ヘッダーやナビゲーションは`__root`に置き、URLごとに変わる部分を`Outlet`に任せます。

次に、トップページです。

```tsx:src/routes/index.tsx
import { createFileRoute, Link } from '@tanstack/react-router';

export const Route = createFileRoute('/')({
  component: Home,
});

function Home() {
  return (
    <section>
      <h1>タスク管理</h1>
      <Link to="/tasks">タスク一覧へ</Link>
    </section>
  );
}
```

`createFileRoute('/')`の引数は、そのファイルが担当するパスです。手で書く必要はありません。ファイルを保存すると、プラグインが正しい値を書き込んでくれます。

:::message
`createFileRoute`の引数を自分で書き換えないでください。ファイルの場所と食い違うと、プラグインが元に戻します。パスを変えたいときは、ファイルを移動またはリネームします。
:::

## Routerを組み立てて描画する

生成されたルートツリーを使って、Routerを作ります。

```tsx:src/main.tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { createRouter, RouterProvider } from '@tanstack/react-router';
import './index.css';
import { routeTree } from './routeTree.gen';

const router = createRouter({
  routeTree,
  defaultPreload: 'intent',
});

declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router;
  }
}

async function enableMocking() {
  if (!import.meta.env.DEV) return;
  const { worker } = await import('./mocks/browser');
  return worker.start({ onUnhandledRequest: 'bypass' });
}

enableMocking().then(() => {
  createRoot(document.getElementById('root')!).render(
    <StrictMode>
      <RouterProvider router={router} />
    </StrictMode>,
  );
});
```

`enableMocking`は「開発環境の準備」の章から引き続き必要です。これを落とすと、APIのモックが起動せず、すべてのリクエストが404になります。

`QueryClientProvider`が消えている点は意図的です。この章はRouter単体の導入に集中します。両方をそろえた形は「QueryとRouterの連携」の章で組み立てます。

`declare module`の部分が、型安全の要です。ここで作ったRouterの型をライブラリに登録しています。この宣言があるおかげで、アプリのどこで`<Link to="...">`と書いても、そのプロジェクトのルート一覧から候補が出て、存在しないパスが弾かれます。

書き忘れると、`to`が単なる文字列として扱われます。動きはしますが、型の恩恵はまったく得られません。導入したのに型が効かないと感じたら、まずここを確認してください。

`defaultPreload: 'intent'`は、リンクにマウスを乗せた時点でそのルートの準備を始める設定です。効果は、Loaderを扱う章で詳しく見ます。

## Linkでリンクを張る

`<a>`の代わりに`<Link>`を使います。

```tsx
<Link to="/tasks">タスク一覧</Link>
<Link to="/tasks/$taskId" params={{ taskId: task.id }}>{task.title}</Link>
```

変数を含むパスでは、`params`で値を渡します。パスに`$taskId`と書いたので、`params`のキーも`taskId`です。ここを`taskID`と書けば型エラーになります。

存在しないパスも同じです。

```tsx
<Link to="/taks">タスク一覧</Link>
//        ^^^^^ 型エラー: そんなルートはない
```

エディタで`to="/`まで打つと、候補の一覧が出ます。ルートの一覧を覚える必要がなくなります。

### 現在地を示す

いま開いているページのリンクは、見た目を変えたいものです。`activeProps`を使います。

```tsx
<Link to="/tasks" activeProps={{ style: { fontWeight: 'bold' } }}>
  タスク
</Link>
```

そのリンクが現在のURLに一致しているときだけ、指定したpropsが適用されます。クラス名を渡す形でも書けます。

```tsx
<Link to="/tasks" activeProps={{ className: 'is-active' }}>タスク</Link>
```

## 動かして確かめる

`npm run dev`で起動します。ヘッダーのリンクで画面が切り替わり、URLが変わります。

画面の隅に、Routerのロゴが増えているはずです。Devtoolsを開くと、次のような情報が見えます。

- 定義されているルートの一覧
- いま合致しているルート
- パスの変数と検索条件の値
- 各ルートの状態（読み込み中、準備済みなど）

Queryのときと同じで、目に見えると理解が速くなります。とくに、あとの章で出てくるLoaderの動きは、このパネルで追うのが手っ取り早いです。

存在しないURL（`/foo`など）を直接入力すると、`__root.tsx`に書いた`notFoundComponent`が表示されます。

## ルートファイルに書けるもの

ここまで`component`と`notFoundComponent`だけを使いました。ルートには、ほかにも多くのオプションを書けます。Routerの学習は、このオプションを1つずつ理解していく作業です。

| オプション | 役割 | 扱う章 |
|---|---|---|
| `component` | そのURLで表示する画面 | この章 |
| `validateSearch` | 検索条件の検証と型付け | 「Search ParamsとURL状態」 |
| `loader` | 遷移と同時に始めるデータ取得 | 「Loaderによるデータ取得」 |
| `loaderDeps` | Loaderが依存する検索条件の宣言 | 「Loaderによるデータ取得」 |
| `pendingComponent` | Loaderの読み込み中の表示 | 「Loaderによるデータ取得」 |
| `errorComponent` | 失敗したときの表示 | 「Loaderによるデータ取得」 |
| `beforeLoad` | 遷移する前に行う検査 | 「認証とアクセス制御」 |

見落とせないのは、これらがすべて**ルートの定義に集まる**ことです。「このURLを開いたら、まず権限を確かめ、データを取り、失敗したらこの画面を出す」という筋書きが、1つのファイルに収まります。コンポーネントの中に散らばりません。

## Code-based Routingとの関係

ファイルを使わず、コードでルートツリーを組む方法もあります。

```tsx
const rootRoute = createRootRoute({ component: RootLayout });
const indexRoute = createRoute({ getParentRoute: () => rootRoute, path: '/', component: Home });
const routeTree = rootRoute.addChildren([indexRoute]);
```

生成ファイルが要らない代わりに、親子関係を自分で書きます。ルートが増えると管理が大変になるため、選ぶ場面は限られます。

ライブラリの内部では、File-based Routingもこの形に変換されています。`routeTree.gen.ts`を開くと、`update`と`addChildren`が並んでいるのが見えます。生成されているのはこのコードです。仕組みを知っておくと、生成ファイルを読むときに戸惑いません。

## まとめ

この章では、TanStack Routerを導入しました。

- 文字列で書くルーティングは、パスやパラメータの間違いを実行時まで検出できません。
- `@tanstack/router-plugin`が`src/routes/`を監視し、`routeTree.gen.ts`を生成します。プラグインは`react()`より前に置きます。
- ファイル名の規則でURLが決まります。`$`が変数、`_`がURLに出ないレイアウト、`-`がルート対象外です。
- 各ルートファイルは`Route`という名前でexportします。`createFileRoute`の引数はプラグインが管理します。
- `__root.tsx`に共通のレイアウトを置き、`<Outlet />`に子を描画します。
- `declare module`でRouterの型を登録します。これを書かないと型が効きません。
- `<Link>`の`to`と`params`が型検査の対象になります。存在しないパスはコンパイルエラーです。
- Devtoolsで、ルートの一覧と現在の状態を確認できます。

次章では、ルートを増やしていきます。パスの変数、ネストしたレイアウト、プログラムからの遷移を扱い、型がどこまで守ってくれるのかを確かめます。
