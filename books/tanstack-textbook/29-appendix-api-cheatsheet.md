---
title: "付録A 目的別API早見表"
---

「やりたいこと」から引ける索引です。詳しい説明は本編の該当章を参照してください。

## TanStack Query

### データを取得する

| 目的 | API・オプション |
|---|---|
| データを取得する | `useQuery` |
| 読み込みの分岐をSuspenseに任せる | `useSuspenseQuery` |
| 数が決まらない並列取得 | `useQueries` |
| 並列取得（Suspense版） | `useSuspenseQueries` |
| 無限スクロール | `useInfiniteQuery` |
| 条件が揃うまで実行しない | `enabled: 条件` |
| 一定間隔で取り直す | `refetchInterval: ミリ秒` |
| 裏に回っても取り直す | `refetchIntervalInBackground: true` |
| データを加工して受け取る | `select: (data) => 加工` |
| 定義を1か所にまとめる | `queryOptions({ ... })` |

### 鮮度とキャッシュ

| 目的 | API・オプション |
|---|---|
| 再取得しない時間を決める | `staleTime: ミリ秒` |
| 自動では古くしない | `staleTime: Infinity` |
| 誰も見ていないデータの寿命 | `gcTime: ミリ秒`（既定5分） |
| マウント時の再取得を止める | `refetchOnMount: false` |
| freshでも必ず取り直す | `refetchOnMount: 'always'` |
| 画面復帰時の再取得を止める | `refetchOnWindowFocus: false` |
| アプリ全体の既定値を決める | `new QueryClient({ defaultOptions: { queries: {...} } })` |

### 状態を読む

| 目的 | 値 |
|---|---|
| データがまだ無い | `isPending` |
| いま通信している | `isFetching` |
| 初回の読み込み中だけ | `isLoading` |
| 失敗した | `isError` / `error` |
| 仮のデータを表示している | `isPlaceholderData` |
| データの有無 | `status`（`pending` / `success` / `error`） |
| 通信の有無 | `fetchStatus`（`fetching` / `paused` / `idle`） |

### データを更新する

| 目的 | API・オプション |
|---|---|
| 更新処理を実行する | `useMutation` |
| 結果を待たずに呼ぶ | `mutate(引数)` |
| 結果を待つ（例外を投げる） | `await mutateAsync(引数)` |
| 成功・失敗・完了時の処理 | `onSuccess` / `onError` / `onSettled` |
| 実行直前の処理（楽観的更新） | `onMutate` |
| 送信中を判定する | `isPending` |
| 送信中の引数を見る | `variables` |

### キャッシュを操作する

| 目的 | API |
|---|---|
| 古い印を付けて再取得する | `queryClient.invalidateQueries({ queryKey })` |
| そのキーだけに絞る | `invalidateQueries({ queryKey, exact: true })` |
| キャッシュを直接書き換える | `queryClient.setQueryData(key, 値または関数)` |
| キャッシュを読む | `queryClient.getQueryData(key)` |
| キャッシュを破棄する | `queryClient.removeQueries({ queryKey })` |
| すべて破棄する（ログアウト） | `queryClient.clear()` |
| 先読みする | `queryClient.prefetchQuery(options)` |
| あることを保証する（Loader用） | `queryClient.ensureQueryData(options)` |
| 進行中の取得を止める | `queryClient.cancelQueries({ queryKey })` |

### エラーと通知

| 目的 | API・オプション |
|---|---|
| 再試行の条件を決める | `retry: (count, error) => boolean` |
| 再試行しない | `retry: false` |
| Error Boundaryへ投げる | `throwOnError: true`（条件も渡せる） |
| 境界のリセットと連動させる | `<QueryErrorResetBoundary>` |
| 全Queryの失敗を拾う | `new QueryCache({ onError })` |
| 全Mutationの失敗を拾う | `new MutationCache({ onError })` |
| 通知対象を選ぶ印を付ける | `meta: { errorMessage: '...' }` |

## TanStack Router

### 移動する

| 目的 | API・オプション |
|---|---|
| リンクを張る | `<Link to="/tasks">` |
| パスの変数を渡す | `params={{ taskId: id }}` |
| 検索条件を渡す | `search={{ page: 2 }}` |
| 現在の条件を元に一部だけ変える | `search={(prev) => ({ ...prev, page: 2 })}` |
| 履歴を積まない | `replace` |
| 現在地のスタイル | `activeProps={{ className: 'is-active' }}` |
| ちょうど一致のときだけ | `activeOptions={{ exact: true }}` |
| プログラムから移動する | `useNavigate()` / `Route.useNavigate()` |
| 1つ前へ戻る | `useRouter().history.back()` |
| 別のルートへ送る（Loader内） | `throw redirect({ to, search })` |

### 値を受け取る

| 目的 | API |
|---|---|
| パスの変数 | `Route.useParams()` |
| 検索条件 | `Route.useSearch()` |
| Loaderの戻り値 | `Route.useLoaderData()` |
| Router Context | `Route.useRouteContext()` |
| ルートファイルの外から | `useParams({ from: 'ルートID' })` |
| どのルートでも使う部品 | `useParams({ strict: false })` |

### ルートの定義

| 目的 | オプション |
|---|---|
| 表示するコンポーネント | `component` |
| 検索条件の検証と型付け | `validateSearch: スキーマ` |
| 検索条件の引き継ぎ | `search: { middlewares: [retainSearchParams([...])] }` |
| 遷移時のデータ取得 | `loader` |
| Loaderの依存を宣言 | `loaderDeps: ({ search }) => ({ ... })` |
| 遷移前の検査 | `beforeLoad` |
| 読み込み中の表示 | `pendingComponent` / `pendingMs` |
| 失敗時の表示 | `errorComponent` |
| データが無い場合 | `notFoundComponent` / `throw notFound()` |
| SSRの制御 | `ssr: true` / `'data-only'` / `false` |
| メタ情報（Start） | `head: () => ({ meta, links })` |

### Routerの設定

| 目的 | オプション |
|---|---|
| 先読みを有効にする | `defaultPreload: 'intent'` |
| 先読みの待ち時間 | `defaultPreloadDelay: 50` |
| Queryと組み合わせる | `defaultPreloadStaleTime: 0` |
| Contextを渡す | `context: { queryClient }` |
| 型を登録する | `declare module '@tanstack/react-router'` |
| スクロール位置の復元 | `scrollRestoration: true` |
| テスト用の履歴 | `history: createMemoryHistory({ initialEntries })` |

## TanStack Table

| 目的 | API・オプション |
|---|---|
| テーブルを作る | `useReactTable` |
| 列を型安全に定義する | `createColumnHelper<T>()` |
| 値を持つ列 | `columnHelper.accessor('key', {...})` |
| 値を持たない列（操作列） | `columnHelper.display({ id, ... })` |
| セルを描画する | `flexRender(cell.column.columnDef.cell, cell.getContext())` |
| 基本のRow Model | `getCoreRowModel()` |
| 並び替えを有効にする | `getSortedRowModel()` |
| 絞り込みを有効にする | `getFilteredRowModel()` |
| ページ区切りを有効にする | `getPaginationRowModel()` |
| 行のIDを指定する | `getRowId: (row) => row.id` |
| 状態を自分で持つ | `state` + `onXxxChange` |
| 初期値だけ決める | `initialState` |
| サーバーに計算を任せる | `manualPagination` / `manualSorting` / `manualFiltering` |
| 総件数を伝える | `rowCount` |
| 並び順を切り替える | `column.getToggleSortingHandler()` |
| 現在の並び順 | `column.getIsSorted()` |
| 列の表示を切り替える | `column.getToggleVisibilityHandler()` |
| 行を選択する | `row.getToggleSelectedHandler()` |
| すべて選択する | `table.getToggleAllRowsSelectedHandler()` |

## TanStack Form

| 目的 | API・オプション |
|---|---|
| フォームを作る | `useForm({ defaultValues, onSubmit })` |
| 定義を共有する | `formOptions({ ... })` |
| 項目を定義する | `<form.Field name="title">{(field) => ...}</form.Field>` |
| 配列の項目 | `<form.Field name="items" mode="array">` |
| 値を変える | `field.handleChange(値)` |
| 触ったことを記録する | `onBlur={field.handleBlur}` |
| 項目の状態 | `field.state.meta`（`isTouched` / `isDirty` / `errors`） |
| 全体の状態を購読する | `<form.Subscribe selector={...}>` |
| 送信できるか | `state.canSubmit` |
| 送信中か | `state.isSubmitting` |
| 送信する | `form.handleSubmit()` |
| 初期値に戻す | `form.reset()` |
| スキーマで検証する | `validators: { onChange: スキーマ }` |
| 表示直後に検証する | `validators: { onMount: スキーマ }` |
| 非同期の検証 | `validators: { onChangeAsync, onChangeAsyncDebounceMs }` |
| サーバー検証を項目に紐づける | `onSubmitAsync`で`{ fields: { 項目名: メッセージ } }`を返す |
| 配列を操作する | `pushValue` / `removeValue` / `insertValue` / `swapValues` / `moveValue` |

## TanStack Virtual

| 目的 | API・オプション |
|---|---|
| 仮想化する | `useVirtualizer({ count, getScrollElement, estimateSize })` |
| 描画する範囲を取得する | `virtualizer.getVirtualItems()` |
| 全体の高さ | `virtualizer.getTotalSize()` |
| 前後に余分に描画する | `overscan: 10` |
| 実際の高さを測る | `ref={virtualizer.measureElement}` + `data-index` |
| 横方向にする | `horizontal: true` |
| 特定の位置へ移動する | `virtualizer.scrollToIndex(index, { align })` |
| 位置を指定して移動する | `virtualizer.scrollToOffset(px)` |

## TanStack Start

| 目的 | API・オプション |
|---|---|
| サーバー処理を定義する | `createServerFn().handler(...)` |
| 更新処理にする | `createServerFn({ method: 'POST' })` |
| 入力を検証する | `.validator(スキーマ)` |
| コンポーネントから呼ぶ | `useServerFn(サーバー関数)` |
| サーバー専用の関数 | `createServerOnlyFn` |
| 環境で実装を分ける | `createIsomorphicFn` |
| APIエンドポイントを作る | `server: { handlers: { GET, POST } }` |
| リクエストを読む | `getRequest()` / `getCookie()` / `getRequestHeader()` |
| レスポンスを操作する | `setResponseStatus()` / `setResponseHeaders()` / `setCookie()` |
| セッションを扱う | `useSession()` / `getSession()` |
| QueryとSSRを統合する | `setupRouterSsrQueryIntegration({ router, queryClient })` |
| HTMLの外枠を書く | `shellComponent` + `<HeadContent />` + `<Scripts />` |
