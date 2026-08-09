---
title: "付録B 他ライブラリからの移行"
---

既存のプロジェクトにTanStackを導入する場合の対応表です。あわせて、古いバージョンの記事を読むときの読み替えもまとめます。

## useEffectによる取得からの移行

もっとも多い出発点です。1つのコンポーネントずつ置き換えられます。

```tsx
// 移行前
const [data, setData] = useState<Task[]>([]);
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState<Error | null>(null);

useEffect(() => {
  setIsLoading(true);
  fetch('/api/tasks')
    .then((res) => res.json())
    .then((result) => setData(result.items))
    .catch(setError)
    .finally(() => setIsLoading(false));
}, []);

// 移行後
const { data, isPending, isError, error } = useQuery({
  queryKey: ['tasks'],
  queryFn: fetchTasks,
});
```

移行の手順を整理します。

1. `QueryClientProvider`をアプリの最上位に置く
2. `fetch`の処理を関数として切り出す（`response.ok`の確認を追加する）
3. 1つのコンポーネントで`useQuery`に置き換える
4. 依存する値（絞り込み条件など）を`queryKey`に入れる
5. `staleTime`を決める

全部を一度に置き換える必要はありません。`useEffect`のままの画面と`useQuery`の画面が混在していても動きます。

## SWRからの移行

考え方が近いため、対応はほぼ1対1です。

| SWR | TanStack Query |
|---|---|
| `useSWR(key, fetcher)` | `useQuery({ queryKey, queryFn })` |
| キーは文字列または配列 | キーは必ず配列 |
| `data` / `error` / `isLoading` | `data` / `error` / `isPending` |
| `mutate()` | `queryClient.invalidateQueries()` |
| `mutate(key, data, false)` | `queryClient.setQueryData(key, data)` |
| `useSWRInfinite` | `useInfiniteQuery` |
| `useSWRMutation` | `useMutation` |
| `revalidateOnFocus` | `refetchOnWindowFocus` |
| `dedupingInterval` | `staleTime` |

いちばん大きな違いは、キーが必ず配列であることと、`staleTime`の既定値です。SWRの`dedupingInterval`は既定2秒ですが、TanStack Queryの`staleTime`は0です。移行するとリクエストが増えたように見えるので、`staleTime`を明示してください。

## Redux / Zustandからサーバー状態を移す

グローバルストアにAPIのデータを入れている場合、その部分だけをTanStack Queryへ移せます。

移行の順番です。

1. ストアの中身を「サーバーのデータ」と「それ以外」に分類する
2. サーバーのデータを`queryOptions`として定義し直す
3. `useSelector`（または`useStore`）を`useQuery`に置き換える
4. データを取得していたAction・Thunk・非同期処理を削除する
5. 更新のActionを`useMutation`に置き換える
6. 残った「それ以外」の状態だけをストアに残す

このとき、ローディングとエラーの状態もストアから消えます。Queryが持つからです。ストアの記述量が大きく減ります。

RTK Queryを使っている場合は、機能が重なります。両方を入れる意味は薄いので、どちらかに寄せる判断が必要です。Reduxを使い続けるならRTK Queryのままでよく、Reduxを外していく方向ならTanStack Queryへ寄せます。

## React Routerからの移行

APIの形が違うため、機械的な置き換えはできません。対応する概念は次のとおりです。

| React Router | TanStack Router |
|---|---|
| `<Route path element>` | `createFileRoute('/path')({ component })` |
| `useParams()` | `Route.useParams()`（型付き） |
| `useSearchParams()` | `Route.useSearch()`（型付き・検証あり） |
| `useNavigate()` | `useNavigate()`（型付き） |
| `<Outlet />` | `<Outlet />`（同じ） |
| `loader` | `loader`（`context`と`deps`が渡る） |
| `useLoaderData()` | `Route.useLoaderData()`（型付き） |
| `errorElement` | `errorComponent` |
| レイアウトルート | `route.tsx` |
| Pathlessなグループ | `_`で始まるファイル |

移行の要点は3つあります。

1つめは、ルートの定義がファイルに移ることです。`createBrowserRouter`の配列を、`src/routes/`のファイルへ分解します。

2つめは、検索条件の扱いが変わることです。`useSearchParams`は文字列を返しますが、TanStack Routerでは`validateSearch`でスキーマを書き、型のついた値を受け取ります。ここが移行で得られるものの中心です。

3つめは、`to`の書き方です。組み立てた文字列（`` `/tasks/${id}` ``）ではなく、パターンと`params`を分けて渡します。

段階的な移行は難しいため、URL構成が小さいうちに切り替えるか、新しい画面から順に作るのが現実的です。

## React Hook Formからの移行

| React Hook Form | TanStack Form |
|---|---|
| `useForm()` | `useForm({ defaultValues })` |
| `register('title')` | `<form.Field name="title">` |
| `<Controller>` | `<form.Field>`（区別がない） |
| `formState.errors` | `field.state.meta.errors` |
| `formState.isDirty` | `state.isDirty` |
| `formState.isSubmitting` | `state.isSubmitting` |
| `handleSubmit(onSubmit)` | `form.handleSubmit()` + `onSubmit`オプション |
| `resolver: zodResolver(schema)` | `validators: { onChange: schema }` |
| `useFieldArray` | `<form.Field mode="array">` |
| `watch('title')` | `<form.Subscribe selector>` |
| `reset(values)` | `form.reset(values)` |

React Hook Formは`register`で入力欄を非制御のまま扱いますが、TanStack Formは`field.state.value`と`handleChange`で制御します。書き方の見た目が変わります。

Zodの`resolver`が要らなくなるのは、Standard Schemaに対応しているためです。`zodResolver`のようなアダプタを挟まず、スキーマをそのまま渡します。

## react-window / react-virtualizedからの移行

| react-window | TanStack Virtual |
|---|---|
| `<FixedSizeList>` | `useVirtualizer({ estimateSize: () => 固定値 })` |
| `<VariableSizeList>` | `useVirtualizer` + `measureElement` |
| `<FixedSizeGrid>` | `useVirtualizer`を縦横に2つ |
| `itemCount` | `count` |
| `itemSize` | `estimateSize` |
| `overscanCount` | `overscan` |
| 子として渡す`({ index, style })` | `getVirtualItems()`を自分で`map`する |

react-windowはコンポーネントを提供しますが、TanStack Virtualは計算だけを返します。マークアップを自分で書くぶん行数は増えますが、`<table>`やグリッドなど任意の構造に適用できます。

## TanStack Query v4以前の記事を読むとき

ネット上には、React Queryという名前だった時代の記事が多く残っています。読み替えの一覧です。

| v4以前 | v5（現行） |
|---|---|
| `react-query` | `@tanstack/react-query` |
| `useQuery(key, fn, options)` | `useQuery({ queryKey, queryFn, ...options })` |
| `isLoading`（データが無い状態） | `isPending` |
| `cacheTime` | `gcTime` |
| `keepPreviousData: true` | `placeholderData: keepPreviousData` |
| `useQuery`のコールバック（`onSuccess`など） | 削除（`useEffect`か`select`で対応） |
| `useErrorBoundary` | `throwOnError` |
| `remove()` | `queryClient.removeQueries()` |
| `refetchPage` | 削除（`maxPages`などで対応） |
| 引数の位置指定 | すべてオブジェクト形式に統一 |

とくに`isLoading`の意味の変化は、はまりやすい部分です。v5の`isLoading`は「初回の読み込み中」を表す別の値として存在します。「データが無い状態」を判定したいなら`isPending`を使ってください。

`useQuery`の`onSuccess`が削除されたことも見落としやすい点です。取得の成功時に副作用を起こす設計自体が推奨されなくなりました。データの加工は`select`、副作用は`useEffect`か、そもそも設計を見直すことになります。

## TanStack Table v7以前の記事を読むとき

| v7以前 | v8（本書の基準） |
|---|---|
| `react-table` | `@tanstack/react-table` |
| `useTable(...)` | `useReactTable({ ... })` |
| `usePagination`などのフック | `getPaginationRowModel()`などのRow Model |
| `columns`に`accessor: 'key'` | `columnHelper.accessor('key', {...})` |
| `Cell: ({ value }) => ...` | `cell: (info) => info.getValue()` |
| `getTableProps()` | 不要（自分でマークアップを書く） |
| `Header` | `header` |

v8はTypeScriptで書き直されており、v7とは別のライブラリに近い変化です。v7向けの記事のコードは、ほぼそのまま使えません。

なお、2026年8月にv9の安定版が公開されました。状態管理がTanStack Storeに載り、機能ごとにコードを取り込む形（tree-shakable）になっています。`createColumnHelper`や`flexRender`といった中心の概念は残るため、v8の知識はそのまま活きます。v8からv9へ移る際は、公式の移行ガイドを参照してください。本書はv8を基準に解説しています。

## TanStack Start の古い記事を読むとき

Startはリリース候補の段階で、構成が何度か変わっています。

| 古い形 | 現行 |
|---|---|
| `@tanstack/start` | `@tanstack/react-start` |
| `app/`ディレクトリ | `src/`ディレクトリ |
| `app/client.tsx`・`app/ssr.tsx`を自分で書く | 不要（プラグインが用意する） |
| `createStartAPIHandler` | ルートの`server.handlers` |
| `routes/api/*.ts`に`createAPIFileRoute` | 通常のルートに`server.handlers` |
| `<Html>` / `<Head>` / `<Body>`コンポーネント | `shellComponent`で`<html>`を直接書く |

Startの記事を読むときは、日付とパッケージ名を最初に確認してください。`@tanstack/start`から始まっているコードは、現行では動きません。
