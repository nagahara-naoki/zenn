---
title: "Entityによるコレクション管理"
---

ここからは、実務設計に入ります。最初のテーマは、コレクションの管理です。

コレクションとは、タスクの一覧のように、同じ種類のデータが並んだものを指します。状態管理では、このコレクションが頻繁に登場します。State設計の章で「コレクションは正規化を検討する」と触れましたが、その正規化を手で書こうとすると、追加・更新・削除のたびに、似たようなコードを何度も書くことになります。この繰り返しを肩代わりしてくれるのが`@ngrx/entity`です。

この章では、`createEntityAdapter`の使い方と、正規化された状態への更新・読み出しを見ていきます。

## なぜEntityを使うのか

まず、Entityがない場合の手間を確認します。State設計の章で見たとおり、コレクションをIDをキーにした平らな形（正規化）で持つと、特定のデータを速く引けます。

```ts
type TasksState = {
  entities: Record<string, Task>; // IDをキーにした本体
  ids: string[]; //                 並び順を保つID配列
};
```

この形は便利ですが、自分で管理しようとすると大変です。タスクを1つ追加するたびに、`entities`に本体を足し、`ids`にもIDを足す。削除するたびに、両方から取り除く。更新するたびに、`entities`の該当する本体を新しくする。しかも、すべてイミュータブルに書かなければなりません。同じようなコードが、Reducerのあちこちに増えていきます。

`@ngrx/entity`は、この`entities`と`ids`の管理を、まるごと引き受けてくれます。追加・更新・削除は、用意された関数を呼ぶだけで済みます。

## createEntityAdapterとEntityState

まず、コレクションを管理する「Adapter（アダプター）」を作ります。`createEntityAdapter`に、扱うデータの型を指定します。

```ts:src/app/tasks/tasks.reducer.ts
import { createEntityAdapter, EntityState } from '@ngrx/entity';

export const adapter = createEntityAdapter<Task>({
  selectId: (task) => task.id, //                          IDの取り出し方
  sortComparer: (a, b) => a.title.localeCompare(b.title), // 並び順
});
```

`selectId`は、「どのプロパティをIDとして使うか」を指定します（省略すると`id`が使われます）。`sortComparer`は、並び順を決める関数です。これを指定すると、`ids`が常にこの順（ここではタイトル順）に保たれます。

状態の型は、`EntityState`を土台にします。コレクション以外の状態（絞り込み条件や通信状態）は、そこに足していきます。

```ts
export type TasksState = EntityState<Task> & {
  filter: Filter;
  selectedTaskId: string | null;
  loading: boolean;
  error: string | null;
};

const initialState: TasksState = adapter.getInitialState({
  filter: 'all',
  selectedTaskId: null,
  loading: false,
  error: null,
});
```

`EntityState<Task>`が、`entities`と`ids`を含む型です。それに`&`で、絞り込みや通信状態の型を足しています。`adapter.getInitialState`は、`entities`と`ids`を空にした初期状態を作り、そこに渡した追加の状態（`filter`など）を混ぜてくれます。

## Adapterの更新関数

Adapterは、コレクションを更新するための関数を持っています。よく使うものを挙げます。

| 関数 | 動き |
|---|---|
| `setAll` | コレクション全体を、渡した配列で置き換える |
| `addOne` / `addMany` | 1件、または複数件を追加する |
| `upsertOne` | あれば更新、なければ追加する |
| `updateOne` | 一部のプロパティだけを更新する |
| `removeOne` / `removeAll` | 削除する |

これらをReducerの中で使います。使い方は、現在の状態を渡すと、更新後の新しい状態を返してくれる、という形です。

```ts:src/app/tasks/tasks.reducer.ts
export const tasksReducer = createReducer(
  initialState,
  // 読み込み成功: 一覧をまるごと入れ替える
  on(tasksActions.loadTasksSuccess, (state, { tasks }) =>
    adapter.setAll(tasks, { ...state, loading: false }),
  ),
  // 追加
  on(tasksActions.addTaskSuccess, (state, { task }) =>
    adapter.addOne(task, state),
  ),
  // 完了状態の切り替え（一部更新）
  on(tasksActions.toggleTaskSuccess, (state, { id, completed }) =>
    adapter.updateOne({ id, changes: { completed } }, state),
  ),
  // 削除
  on(tasksActions.removeTaskSuccess, (state, { id }) =>
    adapter.removeOne(id, state),
  ),
);
```

注目してほしいのは、`adapter.setAll(tasks, { ...state, loading: false })`の部分です。第1引数がコレクションの更新内容、第2引数が対象の状態です。ここでは、一覧を差し替えると同時に、`loading`を`false`にする更新も、まとめて行っています。前章まで自分で書いていた`entities`と`ids`のイミュータブルな組み立ては、すべてAdapterがやってくれます。

`updateOne`には、IDと、変更したいプロパティ（`changes`）を渡します。ここでは`completed`だけを差し替えています。指定したプロパティ以外は、そのまま保たれます。

## Adapterが提供するSelector

Adapterは、更新だけでなく、コレクションを読み出すSelectorも提供してくれます。`getSelectors`で取り出します。

```ts:src/app/tasks/tasks.feature.ts
const { selectAll, selectEntities, selectIds, selectTotal } =
  adapter.getSelectors();
```

| Selector | 返すもの |
|---|---|
| `selectAll` | 全要素の配列（`sortComparer`の順に並ぶ） |
| `selectEntities` | IDをキーにしたオブジェクト |
| `selectIds` | IDの配列 |
| `selectTotal` | 件数 |

これらは、Feature全体の状態を起点にして組み合わせます。`createFeature`と合わせると、次のように書けます。

```ts
export const tasksFeature = createFeature({
  name: 'tasks',
  reducer: tasksReducer,
  extraSelectors: ({ selectTasksState }) => ({
    ...adapter.getSelectors(selectTasksState),
  }),
});

// これで tasksFeature.selectAll などが使えるようになる
```

`selectAll`で並んだ一覧が手に入り、`selectTotal`で件数がすぐに取れます。Selectorの章では、未完了件数を`filter`で手計算していましたが、全体の件数のように基本的なものは、Adapterが最初から提供してくれます。絞り込みのような、さらに込み入った派生値は、これらを入力にしたSelectorを重ねて作ります。

## いつEntityを使うか

`@ngrx/entity`は、便利ですが、どんな状態にも使うものではありません。向いているのは、同じ型のデータが並ぶコレクションです。タスクの一覧、ユーザーの一覧、商品の一覧などです。

逆に、単一のオブジェクトや、少数の固定的な値には、必ずしも必要ありません。たとえば、設定値が数個あるだけの状態に、`entities`と`ids`の仕組みを持ち込むのは、大げさすぎます。判断の目安は、「コレクションで、追加・更新・削除が頻繁に起きるもの」に使う、と考えると分かりやすくなります。

## まとめ

この章では、`@ngrx/entity`によるコレクション管理を確認しました。

- Entityは、正規化された`entities`と`ids`の管理を肩代わりします。
- `createEntityAdapter`でAdapterを作り、`selectId`と`sortComparer`を指定します。
- 状態は`EntityState`を土台にし、`getInitialState`で初期化します。
- `setAll`・`addOne`・`updateOne`・`removeOne`などで、コレクションを更新します。
- `getSelectors`で、`selectAll`・`selectTotal`などのSelectorが手に入ります。
- 追加・更新・削除が頻繁なコレクションに使い、単一の値には使いません。

次章では、ルーティングの状態をNgRxと連携させるRouter Store、開発を助けるDevTools、そしてMeta-Reducerを扱います。
