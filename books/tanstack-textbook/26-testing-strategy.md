---
title: "テスト戦略"
---

ここまで組んだコードをテストします。

TanStackを使ったアプリのテストで最初に決めるべきは、**何をテストしないか**です。ライブラリの動作を検証し始めると、いくらでも書けてしまいます。境界を引いてから、必要なものを書きます。

## 何をテストしないか

TanStack Query、Router、Table、Formには、それぞれ膨大なテストがあります。それらの動作は、ライブラリ側の責任です。

| テストしない | 理由 |
|---|---|
| `staleTime`を過ぎたら再取得されるか | Queryのテストが担保している |
| 前方一致で無効化されるか | 同上 |
| ヘッダーをクリックしたら並び替わるか | Tableのテスト（しかも手動モードでは自分の実装ではない） |
| リンクをクリックしたら遷移するか | Routerのテストが担保している |
| Zodのスキーマが正しく検証するか | Zodのテストが担保している |

書くべきなのは、自分が判断を書いた部分です。

| テストする | 何を確かめるか |
|---|---|
| `queryFn`の実装 | ステータスの確認、レスポンスの変換 |
| Mutation後の無効化 | 更新したら一覧が最新になるか |
| エラー時の表示 | 404と500で違う表示が出るか |
| 検索条件のスキーマ | 壊れた値が既定値に倒れるか |
| フォームの検証と送信 | エラーが出るか、送信できるか |
| 画面の組み立て | データが表示されるか |

判断の目安は「そのコードは自分が書いたか」です。書いたなら壊れる可能性があります。

## テスト環境を作る

VitestとTesting Libraryを使います。

```sh
npm i -D vitest jsdom @testing-library/react @testing-library/user-event @testing-library/jest-dom
```

```ts:vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
  },
});
```

`environment: 'jsdom'`で、Node.jsの中にブラウザに似た環境を用意します。`document`や`window`が使えるようになります。

## モックを再利用する

ここで、「開発環境の準備」の章の投資が回収されます。開発用に書いたMSWのハンドラを、そのままテストで使えます。

```ts:src/test/setup.ts
import '@testing-library/jest-dom/vitest';
import { afterAll, afterEach, beforeAll } from 'vitest';
import { setupServer } from 'msw/node';
import { handlers } from '../mocks/handlers';
import { db } from '../mocks/db';

export const server = setupServer(...handlers);

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));

afterEach(() => {
  server.resetHandlers();
  db.reset();
});

afterAll(() => server.close());
```

ブラウザ用の`setupWorker`が、Node用の`setupServer`に変わっただけです。ハンドラの定義は1文字も変えていません。

3つの後片付けが入っています。

`onUnhandledRequest: 'error'`は、ハンドラのないリクエストが飛んだらテストを失敗させる設定です。開発では`'bypass'`にしていましたが、テストでは厳しくします。意図しない通信に気づけます。

`server.resetHandlers()`は、そのテストだけで差し替えたハンドラを元に戻します。

`db.reset()`は、モックのデータを初期状態に戻します。これを忘れると、作成のテストで増えたタスクが次のテストに影響します。データを初期化できるようにモックを作っておいた理由が、ここにあります。

## テスト用のQueryClient

QueryClientは、テストごとに新しく作ります。そして、既定の設定を変えます。

```tsx:src/test/utils.tsx
export function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        // 失敗を待たずにテストを終えるため、再試行しない
        retry: false,
        // テストの途中でキャッシュを捨てさせない（後片付けのタイマーも動かない）
        gcTime: Infinity,
      },
      mutations: { retry: false },
    },
  });
}

export function renderWithQuery(ui: ReactNode) {
  const queryClient = createTestQueryClient();

  const result = render(<QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>);

  return { ...result, queryClient };
}
```

`retry: false`が必須です。これを忘れると、エラーのテストが3回の再試行を待つことになります。既定の間隔は1秒・2秒・4秒なので、7秒以上かかります。タイムアウトで失敗するか、テスト全体が極端に遅くなります。

QueryClientをテストごとに作るのは、キャッシュを共有しないためです。前のテストで取得したデータが残っていると、「読み込み中」の状態から始まるテストが書けません。

## 一覧のテスト

データが表示されることを確かめます。

```tsx
it('取得したタスクを表示する', async () => {
  renderWithQuery(<TaskList />);

  expect(screen.getByText('読み込み中...')).toBeInTheDocument();

  expect(await screen.findByText(/ログイン画面を設計する/)).toBeInTheDocument();
});
```

最初の`getByText`で読み込み中の表示を確認し、`findByText`でデータの表示を待ちます。

`get`と`find`の違いが要点です。`getBy`は同期的に探し、見つからなければ即座に失敗します。`findBy`はPromiseを返し、見つかるまで待ちます（既定で1秒）。非同期に現れる要素には`findBy`を使います。

## エラーのテスト

そのテストだけハンドラを差し替えます。

```tsx
it('取得に失敗したらエラーを表示する', async () => {
  server.use(
    http.get('/api/tasks', () =>
      HttpResponse.json({ message: 'サーバーエラー' }, { status: 500 }),
    ),
  );

  renderWithQuery(<TaskList />);

  expect(await screen.findByText(/サーバーエラー/)).toBeInTheDocument();
});
```

`server.use()`で、そのテストの間だけハンドラを上書きします。`afterEach`の`resetHandlers()`で元に戻るので、他のテストには影響しません。

404と500で表示を分けている場合は、ステータスを変えたテストを2本書きます。エラー処理は分岐が多いので、テストの価値が高い部分です。

## 重複排除のテスト

「同じ`queryKey`なら1回しか取得しない」という振る舞いは、ライブラリの機能です。ただ、自分の実装で`queryKey`がずれていないかを確かめる意味はあります。

```tsx
it('同じqueryKeyのコンポーネントは1回の取得を共有する', async () => {
  let calls = 0;
  server.use(
    http.get('/api/tasks', () => {
      calls += 1;
      return HttpResponse.json({ items: [], total: 0, page: 1, perPage: 20 });
    }),
  );

  renderWithQuery(
    <>
      <TaskCount />
      <TaskList />
    </>,
  );

  await waitFor(() => expect(screen.getByText('全0件')).toBeInTheDocument());
  expect(calls).toBe(1);
});
```

リクエストの回数を数えています。もし`TaskCount`と`TaskList`で`queryKey`の書き方が違っていたら、`calls`が2になって失敗します。キーの取り違えを検出できるテストです。

## Mutationのテスト

更新したら一覧が最新になる、という一連の流れを確かめます。

```tsx
it('作成すると一覧が最新化される', async () => {
  const user = userEvent.setup();
  renderWithQuery(
    <>
      <CreateTaskForm />
      <TaskCount />
    </>,
  );

  await waitFor(() => expect(screen.getByText('全137件')).toBeInTheDocument());

  await user.type(screen.getByRole('textbox'), '新しいタスク');
  await user.click(screen.getByRole('button', { name: '作成' }));

  await waitFor(() => expect(screen.getByText('全138件')).toBeInTheDocument());
});
```

件数が137から138に変わることを確認しています。この1本で、Mutationの実行、キャッシュの無効化、再取得、再描画までが通ります。

`invalidateQueries`のキーを書き間違えていれば、件数が変わらずに失敗します。「更新したのに画面が変わらない」という、実際によくあるバグを捕まえられるテストです。

:::message
要素の取得には、`getByRole`を優先してください。`getByTestId`はマークアップの変更に弱く、`getByText`は文言の変更に弱いという弱点があります。

`getByRole('button', { name: '作成' })`は、「作成というラベルのボタン」を探します。これは、スクリーンリーダーが読み上げる情報と同じです。テストがアクセシビリティの確認も兼ねます。ボタンに適切なラベルが無ければ、テストが書けないことで気づけます。
:::

## フォームのテスト

検証のテストは、入力してから確かめます。

```tsx
it('入力が短すぎるとエラーを表示する', async () => {
  const user = userEvent.setup();
  renderWithQuery(<TaskForm />);

  const title = screen.getByLabelText('タイトル');
  await user.type(title, 'a');
  await user.clear(title);
  await user.tab();

  expect(await screen.findByRole('alert')).toHaveTextContent('タイトルは必須です');
});
```

`user.tab()`でフォーカスを外しています。`isTouched`が`true`にならないとエラーが表示されない実装なので、この操作が必要です。

`userEvent`は、実際のユーザー操作に近い形でイベントを発生させます。`fireEvent`より遅いものの、フォーカスやキー入力の順序まで再現してくれます。フォームのテストでは`userEvent`を選んでください。

:::message alert
このテストを書いていて発見したことがあります。フォームを表示した直後、まだ何も入力していない状態で、送信ボタンが**押せる状態**になっていました。

`canSubmit`は検証を通っているかを表しますが、`onChange`の検証は値が変わらないと走りません。初期状態では検証が1度も実行されていないため、通っている扱いになります。

`validators`に`onMount`を追加すると、表示時に検証が走って`canSubmit`が`false`になります。テストを書いたことで、実装の穴が見つかった例です。
:::

## ルートのテスト

ルートのテストは、Routerを組み立てる手間がかかります。

```tsx
import { createMemoryHistory, createRouter, RouterProvider } from '@tanstack/react-router';
import { routeTree } from '../routeTree.gen';

function renderRoute(initialPath: string) {
  const queryClient = createTestQueryClient();
  const router = createRouter({
    routeTree,
    context: { queryClient },
    history: createMemoryHistory({ initialEntries: [initialPath] }),
  });

  return render(
    <QueryClientProvider client={queryClient}>
      <RouterProvider router={router} />
    </QueryClientProvider>,
  );
}
```

`createMemoryHistory`で、ブラウザの履歴を使わずに初期URLを指定します。これで「`/tasks?page=2`を開いたとき」のテストが書けます。

ただ、ルートのテストは重くなりがちです。ルートツリー全体を読み込み、Loaderが動き、認証の`beforeLoad`まで走ります。

現実的な方針は、次のように分けることです。

- 画面の中身のテストは、コンポーネント単体で書く（Routerを使わない）
- URLに関わる判断（リダイレクト、検索条件の検証）だけ、Routerを組み立ててテストする
- 画面をまたぐ流れは、E2Eテストに任せる

そのために、コンポーネントをルートファイルから切り出しておくと有利です。`routes/`のファイルを薄く保つ理由の1つです。

## テーブルのテスト

サーバーサイド処理のテーブルでは、並び替えや絞り込みの計算をライブラリが行いません。テストすべきは、**操作がURLの変更に正しく変換されるか**です。

```tsx
// 疑似コード
await user.click(screen.getByRole('button', { name: /期限/ }));
expect(router.state.location.search).toMatchObject({ sort: 'dueDate', order: 'desc', page: 1 });
```

行の描画そのものは、データを渡して表示されることを確認すれば足ります。セルのカスタム表示（バッジや日付の書式）は、自分が書いたコードなのでテストの価値があります。

## E2Eテストとの分担

ここまでのテストは、すべてjsdomの中で動いています。本物のブラウザではありません。

jsdomで確認しづらいものがあります。

- 実際のスクロール（仮想化の動作）
- Service Workerの挙動
- 複数タブ間の同期
- CSSによる表示崩れ
- 実際のフォーカスの移動

これらはPlaywrightのようなE2Eテストの領域です。役割を分けます。

| 層 | 担当 |
|---|---|
| 単体・結合（Vitest + jsdom） | 状態の変化、検証、表示の分岐、キャッシュの連携 |
| E2E（Playwrightなど） | 主要な業務フロー、ブラウザ固有の挙動 |

E2Eは実行が遅く、壊れやすいものです。「ログインしてタスクを作成して一覧で確認する」のような、収益に直結する数本に絞るのが現実的です。細かい分岐は、速く動く単体テストで網羅します。

## まとめ

ライブラリの内部ではなく、利用者から見える振る舞いをテストすれば、実装を変えても意図を守れます。

- ライブラリの動作はテストしません。自分が判断を書いた部分だけをテストします。
- MSWのハンドラを開発とテストで共有します。`setupWorker`が`setupServer`になるだけです。
- テストでは`onUnhandledRequest: 'error'`にして、意図しない通信を検出します。
- `afterEach`でハンドラとモックのデータを初期化します。
- テスト用のQueryClientは`retry: false`にします。忘れると、エラーのテストが再試行を待って極端に遅くなります。
- QueryClientはテストごとに作り、キャッシュを共有しません。
- 非同期に現れる要素は`findBy`、すでにある要素は`getBy`で取得します。
- `server.use()`でそのテストだけハンドラを差し替え、エラー時の表示を確認します。
- リクエストの回数を数えると、`queryKey`の取り違えを検出できます。
- 件数の変化を確認するテスト1本で、Mutationから再取得までの流れを通せます。
- 要素の取得は`getByRole`を優先します。テストがアクセシビリティの確認も兼ねます。
- ルートのテストは`createMemoryHistory`で初期URLを指定します。重いので、URLに関わる判断に絞ります。
- jsdomで確認しづらいものはE2Eに任せ、本数を絞ります。

次章から最終部です。TanStack Startを導入し、ここまでクライアントだけで動かしてきたアプリを、サーバー側へ広げます。
