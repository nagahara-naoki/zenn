---
title: "ReducerとSelectorをテストする"
---

ここからは、テストに入ります。NgRxの大きな利点の1つが、テストのしやすさです。

なぜテストしやすいのか。その理由は、これまで純粋関数にこだわってきたことにあります。Reducerは純粋関数で、Selectorの計算ロジック（projector）も純粋関数です。純粋関数は、入力を渡して、返ってきた出力を確かめるだけでテストできます。Storeを起動する必要も、通信の完了を待つ必要も、コンポーネントを描画する必要もありません。この身軽さが、NgRxのテストを楽にしています。

この章では、ReducerとSelectorのテストを扱います。テストフレームワークは、Angularの現行世代で標準のVitestを前提としますが、書き方の考え方は、どのフレームワークでも同じです。

## 純粋関数はなぜテストしやすいのか

まず、テストしやすさの理由を、もう少し丁寧に押さえます。

Reducerは、いまの状態とActionを受け取り、新しい状態を返す純粋関数でした。ここで「純粋」であることが効いてきます。純粋関数は、外部の状態に依存せず、副作用も持ちません。だから、テストのために用意するのは「入力（状態とAction）」だけで、モックやセットアップはほとんど要りません。しかも、同じ入力からは常に同じ出力が返るので、「たまに失敗する」ような不安定なテストになりません。状態管理のロジックを、外の世界から切り離して、単体で検証できるのです。

## Reducerをテストする

Reducerのテストは、初期状態とActionを渡して、期待どおりの状態が返るかを確かめる、という形です。

```ts:src/app/tasks/tasks.reducer.spec.ts
import { describe, it, expect } from 'vitest';
import { tasksReducer } from './tasks.reducer';
import { tasksActions } from './tasks.actions';

describe('tasksReducer', () => {
  const initialState = {
    tasks: [],
    filter: 'all' as const,
    loading: false,
    error: null,
  };

  it('setFilterで絞り込み条件が変わる', () => {
    const next = tasksReducer(initialState, tasksActions.setFilter({ filter: 'active' }));
    expect(next.filter).toBe('active');
  });

  it('loadTasksSuccessで一覧が入り、loadingが下りる', () => {
    const loading = { ...initialState, loading: true };
    const tasks = [{ id: '1', title: 'A', completed: false }];

    const next = tasksReducer(loading, tasksActions.loadTasksSuccess({ tasks }));

    expect(next.tasks).toEqual(tasks);
    expect(next.loading).toBe(false);
  });
});
```

やっていることは単純です。Reducerを、ただの関数として直接呼び、返ってきた状態を`expect`で検証しているだけです。NgRxのStoreも、`provideStore`も、いっさい起動していません。Reducerが純粋関数だからこそ、このシンプルなテストが成り立ちます。2つ目のテストでは、`loading: true`の状態から始めて、`loadTasksSuccess`で一覧が入り、`loading`が`false`に下りることを、まとめて確かめています。

## イミュータブルを検証する

Reducerが「元の状態を書き換えていない」ことも、テストで確かめられます。更新後の状態が、元の状態とは別のオブジェクトになっていることを検証します。

```ts
it('元の状態を書き換えない', () => {
  const next = tasksReducer(initialState, tasksActions.setFilter({ filter: 'active' }));

  expect(next).not.toBe(initialState); // 別のオブジェクトである
  expect(initialState.filter).toBe('all'); // 元は変わっていない
});
```

`not.toBe(initialState)`で、返ってきた状態が元とは別のオブジェクト（参照が違う）ことを確かめます。さらに、`initialState.filter`が`'all'`のままであることも確認しています。もしReducerが元の状態を直接書き換えていたら、この`initialState.filter`は`'active'`に変わってしまうはずです。こうしたテストを1つ置いておくと、イミュータブルな更新が守られているかを、機械的に保証できます。

## Selectorをテストする

Selectorのテストは、計算ロジックであるprojectorを直接呼びます。Selectorの章で触れた`projector`です。ここでも、Storeも状態全体も用意せず、入力だけを直接渡します。

```ts:src/app/tasks/tasks.selectors.spec.ts
import { describe, it, expect } from 'vitest';
import { selectVisibleTasks } from './tasks.selectors';

describe('selectVisibleTasks', () => {
  const tasks = [
    { id: '1', title: 'A', completed: false },
    { id: '2', title: 'B', completed: true },
  ];

  it('activeのとき未完了だけを返す', () => {
    const result = selectVisibleTasks.projector(tasks, 'active');
    expect(result).toEqual([{ id: '1', title: 'A', completed: false }]);
  });

  it('completedのとき完了だけを返す', () => {
    const result = selectVisibleTasks.projector(tasks, 'completed');
    expect(result).toEqual([{ id: '2', title: 'B', completed: true }]);
  });
});
```

`selectVisibleTasks.projector(tasks, filter)`と書くと、Selectorの計算関数だけを取り出して呼べます。状態全体を組み立てる必要はなく、入力（一覧と絞り込み条件）を直接渡し、出力（絞り込み後の一覧）を確かめるだけです。`active`なら未完了だけ、`completed`なら完了だけが返る、という絞り込みのロジックを、ピンポイントで検証しています。

## ViewModelのSelectorをテストする

ViewModelの章で作った、複数の値をまとめるSelectorも、同じようにprojectorでテストできます。画面の表示ロジックがprojectorに集まっているので、そこを検証すれば、画面が正しい値を組み立てられるかを確かめられます。

```ts
it('ViewModelが必要な値をまとめて返す', () => {
  const vm = selectTaskListViewModel.projector(
    tasks, //  visibleTasks
    1, //      incompleteCount
    false, //  loading
    null, //   error
    'all', //  filter
  );

  expect(vm.incompleteCount).toBe(1);
  expect(vm.loading).toBe(false);
});
```

コンポーネントを起動しなくても、表示ロジックだけを単体でテストできています。ViewModelパターンには、コンポーネントを薄くするだけでなく、こうしてテストしやすくなる、という利点もあったわけです。

## テストで設計を確かめる

ReducerとSelectorが、これほど簡単にテストできるのは、偶然ではありません。これまでの設計の積み重ねの結果です。副作用をEffectsに追い出し、派生値をSelectorに寄せ、状態をイミュータブルに更新してきたからこそ、ロジックが純粋関数に集まり、単体で検証できるのです。

このことは、逆向きにも使えます。もし「テストが書きにくい」と感じたら、それは設計を見直す合図かもしれません。Reducerに副作用が混じっていないか。コンポーネントに、本来Selectorへ寄せるべき表示ロジックが残っていないか。テストのしやすさは、設計が健全かどうかを映す鏡になります。書きにくさを感じたら、まず設計を疑ってみてください。

## まとめ

この章では、ReducerとSelectorのテストを確認しました。

- ReducerとSelectorは純粋関数なので、入力と出力だけでテストできます。
- Reducerは、状態とActionを渡し、返る状態を検証します。Storeは要りません。
- 更新後が別オブジェクトであることを確かめ、イミュータブルを保証できます。
- Selectorは`projector`を直接呼び、計算ロジックを単体でテストします。
- ViewModelのSelectorも、コンポーネントなしで表示ロジックを検証できます。
- テストのしやすさは、副作用の分離や派生値の集約といった、設計の結果です。

次章では、副作用をともなうEffects、そしてComponentStore・SignalStoreのテストを扱います。
