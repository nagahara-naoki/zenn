---
title: "Selectorを組み合わせてViewModelを作る"
---

前章で、Selectorのメモ化を学びました。この章では、複数のSelectorを組み合わせて、画面が必要とする形の値を1つにまとめる方法を扱います。

コンポーネントが状態を読み出すとき、いくつものSelectorを個別に受け取り、テンプレートの中で組み立てていくと、コンポーネントがどんどん太っていきます。そのかわりに、画面が必要とするものをあらかじめ1つのSelectorにまとめておけば、コンポーネントは薄く保てます。この「まとめたもの」を、ViewModel（表示のためのモデル）と呼びます。

Selectorの合成、ViewModelパターン、そして実行時の値で絞り込むSelectorの書き方を、順に見ていきます。

## Selectorを合成する

前章で少し触れたとおり、`createSelector`は、入力としてほかのSelectorを取れます。この性質を使うと、Selectorを段階的に積み上げられます。

たとえば、前章で作った`selectIncompleteCount`（未完了件数）を入力にして、さらに別のSelectorを作れます。

```ts
export const selectHasIncompleteTasks = createSelector(
  selectIncompleteCount,
  (count) => count > 0,
);
```

`selectHasIncompleteTasks`は、「未完了のタスクが1件でもあるか」を返します。件数を数える`selectIncompleteCount`の結果を受け取って、`> 0`かどうかを判定しているだけです。小さなSelectorを組み合わせて、より大きなSelectorを作る。しかも、それぞれがメモ化されるので、計算は必要なときだけ走ります。Selectorは、こうしてレゴブロックのように積み重ねられます。

## ViewModelパターン

コンポーネントが必要とする値が増えてくると、Selectorを個別に読み出す宣言が、ずらりと並びます。

```ts
// 個別に読み出すと、コンポーネントに宣言が並ぶ
readonly tasks = this.store.selectSignal(selectVisibleTasks);
readonly count = this.store.selectSignal(selectIncompleteCount);
readonly loading = this.store.selectSignal(tasksFeature.selectLoading);
readonly error = this.store.selectSignal(tasksFeature.selectError);
readonly filter = this.store.selectSignal(tasksFeature.selectFilter);
```

5つのSelectorを別々に読んでいます。動きはしますが、コンポーネントが状態の詳細を知りすぎている感じがします。

そこで、これらを1つのSelectorにまとめてしまいます。画面に必要な情報をひとまとめにしたこの値が、ViewModelです。

```ts
export const selectTaskListViewModel = createSelector(
  selectVisibleTasks,
  selectIncompleteCount,
  tasksFeature.selectLoading,
  tasksFeature.selectError,
  tasksFeature.selectFilter,
  (tasks, incompleteCount, loading, error, filter) => ({
    tasks,
    incompleteCount,
    loading,
    error,
    filter,
  }),
);
```

5つのSelectorを入力に取り、それらを1つのオブジェクトにまとめて返しています。すると、コンポーネントは、このViewModelを1つ読むだけで済みます。

```ts
@Component({ /* ... */ })
export class TaskListComponent {
  private readonly store = inject(Store);
  readonly vm = this.store.selectSignal(selectTaskListViewModel);
}
```

```html
@if (vm().loading) {
  <p>読み込み中...</p>
} @else {
  <p>未完了: {{ vm().incompleteCount }}件</p>
  @for (task of vm().tasks; track task.id) {
    <li>{{ task.title }}</li>
  }
}
```

コンポーネントには、状態を組み立てるロジックが1つもありません。何をどう表示するかはSelectorが決め、コンポーネントは`vm()`を受け取って、そのまま画面に映すだけです。役割がきれいに分かれます。

## ViewModelの利点

ViewModelパターンには、いくつかの利点があります。

- **コンポーネントが薄くなる**: 表示に必要な計算がSelectorへ移り、コンポーネントは描画に専念できます。
- **テストしやすい**: 画面のロジックがprojectorに集まるので、そこだけを純粋関数として単体テストできます。
- **メモ化が効く**: ViewModel自体もメモ化されるので、入力が変わらなければ再計算されません。

ここで、初学者が抱きやすい疑問に答えておきます。「ViewModelは毎回新しいオブジェクトを作っているように見えるが、無駄ではないか」という疑問です。たしかにprojectorはオブジェクトを作りますが、メモ化のおかげで、入力のSelectorがどれも変わらなければ、projectorは呼ばれず、前回と同じViewModelが返ります。つまり、無駄な再計算は起きません。前章のメモ化の知識が、ここで安心材料になります。

## 引数を受け取るSelector

画面によっては、実行時に決まる値でSelectorを絞りたいことがあります。たとえば「指定したIDのタスクを取り出す」場合です。IDは、実行してみないと決まりません。

このようなときは、Selectorを返す関数（ファクトリ）として書きます。

```ts
export const selectTaskById = (id: string) =>
  createSelector(
    tasksFeature.selectTasks,
    (tasks) => tasks.find((task) => task.id === id) ?? null,
  );
```

`selectTaskById`は、IDを受け取って、そのIDに対応するSelectorを作って返す関数です。呼び出すときは、IDを渡してSelectorを作り、それを読み出します。

```ts
readonly task = this.store.selectSignal(selectTaskById(this.taskId));
```

補足として、古いNgRxには、Selectorに`props`という引数を渡す書き方がありました。現在それは非推奨で、このファクトリ関数の形が推奨されています。ただし、ファクトリは呼ぶたびに新しいSelectorを作るため、メモ化が呼び出しをまたいでは効かない、という注意点があります。同じIDで繰り返し使うなら、作ったSelectorを変数に保持して使い回します。

## どこまでをSelectorに寄せるか

ViewModelは便利ですが、何でもかんでもSelectorに詰め込むと、今度はSelectorが太ってしまいます。境界の目安を持っておきましょう。

目安はこうです。状態から純粋に計算できる「表示のためのロジック」はSelectorへ。ユーザー操作への反応やイベント処理はコンポーネントへ。この線で分けます。

Selectorが担うのは、「状態を、画面が読みたい形に変換する」ところまでです。状態を更新するのはActionとReducer、副作用はEffects、という役割分担は、ViewModelを作るときも変わりません。この線引きを意識しておくと、「この処理はどこに書くべきか」で迷わなくなります。

## まとめ

この章では、Selectorの合成とViewModelパターンを確認しました。

- `createSelector`は入力にSelectorを取れるので、段階的に組み立てられます。
- 画面に必要な値を1つにまとめたSelectorを、ViewModelと呼びます。
- ViewModelを使うと、コンポーネントが薄くなり、テストもしやすくなります。
- メモ化が効くため、ViewModelがオブジェクトを返しても無駄な再計算は起きません。
- 実行時の値で絞るSelectorは、Selectorを返すファクトリ関数として書きます。
- 状態から計算できる表示ロジックはSelectorへ、操作への反応はコンポーネントへ分けます。

次章からは、副作用を扱うEffectsに入ります。まず、Effectsの基本から始めます。
