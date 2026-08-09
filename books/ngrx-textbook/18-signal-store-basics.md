---
title: "SignalStoreの基本"
---

ここからは、Signalsをベースにした状態管理、SignalStoreに入ります。

ここまで学んできたClassic Store（Action・Reducer・Selector・Effects）は、大規模で、予測可能性が重要なアプリに向いています。変更がすべてActionとして記録され、DevToolsで追える、という強みがありました。しかし、その仕組みは記述量が多く、小さな状態にはやや重く感じられます。Actionを定義し、Reducerを書き、Selectorを作り……という手順は、ちょっとした状態を扱うには大げさなことがあります。

SignalStoreは、AngularのSignalsを土台に、ずっと少ない記述で状態管理を実現する仕組みです。この章では、`signalStore`の基本、状態がSignalになること、派生値と更新メソッドの書き方を見ていきます。これまでのタスク管理の状態を、SignalStoreで組み立て直してみましょう。

## SignalStoreとは

SignalStoreは、`@ngrx/signals`が提供する、Signalsベースの状態管理です。状態をSignalとして持ち、派生値と更新メソッドを、1つのStoreにまとめます。

Classic Storeとのいちばんの違いは、ActionやReducerを介さないことです。Classic Storeでは、状態を変えるのに「Actionを発行してReducerを通す」という手順が必要でした。SignalStoreでは、Storeのメソッドを呼んで、状態を直接更新します。そのぶん記述が減り、1つの機能や1つのコンポーネントの状態を、手軽に管理できます。単方向データフローの厳密さよりも、簡潔さと開発の速さを重視したい場面に向いています。

## signalStoreとwithState

SignalStoreは、`signalStore`という関数に、「フィーチャ」と呼ばれる部品を並べて作ります。まず、状態を定義するフィーチャが`withState`です。

```ts:src/app/tasks/tasks.store.ts
import { signalStore, withState } from '@ngrx/signals';

type Filter = 'all' | 'active' | 'completed';

type TasksState = {
  tasks: Task[];
  filter: Filter;
  loading: boolean;
};

export const TasksStore = signalStore(
  { providedIn: 'root' },
  withState<TasksState>({
    tasks: [],
    filter: 'all',
    loading: false,
  }),
);
```

`withState`に初期状態を渡すと、その各プロパティ（`tasks`・`filter`・`loading`）が、Storeの上でそれぞれSignalになります。`{ providedIn: 'root' }`を指定すると、アプリ全体で共有される1つのStoreになります。この指定を省くと、コンポーネントの`providers`で提供する、そのコンポーネント専用のStoreになります。全体で使うか、画面に閉じるかを、ここで選べます。

## 状態はSignalになる

`withState`で定義した状態は、Signalとして読み出せます。コンポーネントでStoreを注入し、Signalを呼ぶだけです。

```ts
import { Component, inject } from '@angular/core';
import { TasksStore } from './tasks.store';

@Component({ /* ... */ })
export class TaskListComponent {
  readonly store = inject(TasksStore);
  // store.tasks() でタスク一覧、store.filter() で絞り込み条件が読める
}
```

```html
<p>絞り込み: {{ store.filter() }}</p>
@for (task of store.tasks(); track task.id) {
  <li>{{ task.title }}</li>
}
```

ここが、Classic Storeとの体験の違いです。Classic Storeでは、Selectorを作り、`selectSignal`で読み出していました。SignalStoreでは、`store.tasks()`のように、Storeのプロパティを直接呼ぶだけで読めます。状態がそのままSignalになっているので、Angularのテンプレートと、余計な手順なしにつながります。

## withComputedで派生値を作る

派生値は、`withComputed`というフィーチャで定義します。状態のSignalを受け取り、Angularの`computed`で計算した、新しいSignalを返します。

```ts:src/app/tasks/tasks.store.ts
import { signalStore, withState, withComputed } from '@ngrx/signals';
import { computed } from '@angular/core';

export const TasksStore = signalStore(
  { providedIn: 'root' },
  withState<TasksState>({ tasks: [], filter: 'all', loading: false }),
  withComputed(({ tasks, filter }) => ({
    visibleTasks: computed(() => {
      switch (filter()) {
        case 'active':
          return tasks().filter((task) => !task.completed);
        case 'completed':
          return tasks().filter((task) => task.completed);
        default:
          return tasks();
      }
    }),
    incompleteCount: computed(() => tasks().filter((t) => !t.completed).length),
  })),
);
```

`withComputed`は、状態のSignal（`tasks`や`filter`）を受け取ります。それらを使って`computed`で派生値を作ると、元のSignalが変わったときだけ再計算されます。これは、Classic StoreのSelectorが持っていたメモ化と、まったく同じ働きです。仕組みは違っても、「派生値は元が変わったときだけ計算し直す」という考え方は共通です。そして「派生値は状態に持たず、計算で導く」という原則も、SignalStoreでそのまま生きています。`visibleTasks`も`incompleteCount`も、`withState`には持たず、`withComputed`で導いています。

## withMethodsで更新する

状態を更新するメソッドは、`withMethods`で定義します。更新には`patchState`を使います。

```ts:src/app/tasks/tasks.store.ts
import {
  signalStore,
  withState,
  withComputed,
  withMethods,
  patchState,
} from '@ngrx/signals';

export const TasksStore = signalStore(
  { providedIn: 'root' },
  withState<TasksState>({ tasks: [], filter: 'all', loading: false }),
  withComputed(/* ... */),
  withMethods((store) => ({
    setFilter(filter: Filter) {
      patchState(store, { filter });
    },
    addTask(title: string) {
      patchState(store, {
        tasks: [
          ...store.tasks(),
          { id: crypto.randomUUID(), title, completed: false },
        ],
      });
    },
    toggleTask(id: string) {
      patchState(store, {
        tasks: store.tasks().map((task) =>
          task.id === id ? { ...task, completed: !task.completed } : task,
        ),
      });
    },
  })),
);
```

`patchState(store, 変更分)`で、状態の一部を更新します。渡した分だけが差し替わり、残りはそのまま保たれます。ここで注目してほしいのは、更新の中身が、Reducerで書いたイミュータブルな更新と、まったく同じだということです。配列は`map`やスプレッドで新しく作っています。Reducerで学んだ書き方が、そのまま通用します。

`patchState`には、現在の状態を受け取る関数も渡せます。

```ts
patchState(store, (state) => ({ tasks: [...state.tasks, newTask] }));
```

## コンポーネントから使う

Storeを注入し、Signalで状態を読み、メソッドで更新します。

```ts
@Component({ /* ... */ })
export class TaskListComponent {
  readonly store = inject(TasksStore);

  onAdd(title: string) {
    this.store.addTask(title);
  }
}
```

```html
<p>未完了: {{ store.incompleteCount() }}件</p>
@for (task of store.visibleTasks(); track task.id) {
  <li (click)="store.toggleTask(task.id)">{{ task.title }}</li>
}
```

流れを、Classic Storeと比べてみましょう。Classic Storeでは、Actionをdispatchし、Reducerが更新し、Selectorで読み出しました。SignalStoreでは、メソッドを呼び、`patchState`が更新し、Signalで読み出します。登場人物が減り、状態の定義から利用までが、1つのStoreの中に収まっています。

## Classic Storeとの違い

ここまでを、Classic Storeと並べて整理します。

| 観点 | Classic Store | SignalStore |
|---|---|---|
| 状態の読み出し | Selector | 状態のSignal |
| 派生値 | Selector（メモ化） | `withComputed`（`computed`） |
| 更新 | Action → Reducer | メソッド＋`patchState` |
| 記述量 | 多い | 少ない |
| 変更の追跡 | DevToolsで詳細に追える | 直接更新のため追いにくい |

表を見ると、SignalStoreの手軽さがよくわかります。ただし、良いことばかりではありません。SignalStoreは更新が直接的なぶん、Classic StoreのようにはActionの履歴で追えません。「いつ、何がきっかけで状態が変わったか」を厳密に追いたい大規模アプリでは、Classic Storeの予測可能性が生きます。どちらを選ぶかは、アプリの規模と、予測可能性への要求で決まります。この使い分けは、本書の最終章で詳しく扱います。

## まとめ

この章では、SignalStoreの基本を確認しました。

- SignalStoreは、Signalsをベースにした、記述の少ない状態管理です。
- `signalStore`に`withState`を並べ、状態の各プロパティがSignalになります。
- `{ providedIn: 'root' }`で全体共有、コンポーネント提供で画面専用のStoreになります。
- `withComputed`と`computed`で派生値を作り、元が変わったときだけ再計算されます。
- `withMethods`でメソッドを定義し、`patchState`でイミュータブルに更新します。
- Classic Storeより手軽ですが、変更の追跡は直接的で、追いにくくなります。

次章では、SignalStoreの非同期処理と拡張を扱います。`rxMethod`によるAPI通信や、`withEntities`などの拡張を見ていきます。
