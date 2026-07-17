---
title: "SignalStoreの非同期処理と拡張"
---

前章で、SignalStoreの基本を組み立てました。この章では、実務で必要になる非同期処理と、Storeを拡張する仕組みを扱います。

Classic Storeでは、API通信などの副作用をEffectsが担いました。では、SignalStoreではどうするのでしょうか。その役割を引き受けるのが`rxMethod`です。あわせて、コレクションを扱う`withEntities`、Storeの初期化・破棄を扱う`withHooks`、そして機能を再利用する`signalStoreFeature`を見ていきます。

前章のタスク管理のStoreを、API通信に対応させながら、少しずつ拡張していきます。

## rxMethodで非同期処理を書く

SignalStoreで非同期処理を書くには、`@ngrx/signals/rxjs-interop`が提供する`rxMethod`を使います。名前のとおり、RxJSのストリームを、Storeのメソッドとして定義できる仕組みです。

```ts:src/app/tasks/tasks.store.ts
import { signalStore, withState, withMethods, patchState } from '@ngrx/signals';
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { tapResponse } from '@ngrx/operators';
import { pipe, switchMap, tap } from 'rxjs';
import { inject } from '@angular/core';

export const TasksStore = signalStore(
  { providedIn: 'root' },
  withState<TasksState>({ tasks: [], filter: 'all', loading: false, error: null }),
  withMethods((store, api = inject(TaskApi)) => ({
    loadTasks: rxMethod<void>(
      pipe(
        tap(() => patchState(store, { loading: true, error: null })),
        switchMap(() =>
          api.getTasks().pipe(
            tapResponse({
              next: (tasks) => patchState(store, { tasks, loading: false }),
              error: (error: Error) =>
                patchState(store, { loading: false, error: error.message }),
            }),
          ),
        ),
      ),
    ),
  })),
);
```

流れを追いましょう。`rxMethod`には、RxJSの`pipe`で組んだストリームを渡します。まず`tap`で`loading`を立て、`switchMap`でAPIを呼び、`tapResponse`で成功・失敗を処理します。成功なら`patchState`で一覧を入れて`loading`を下ろし、失敗ならエラーを入れます。

見比べてほしいのは、EffectsのAPI通信とよく似ている点です。`switchMap`でAPIを呼ぶところ、成功・失敗で処理を分けるところは、考え方が同じです。Effectsの章で学んだFlattening Operatorの選択（読み込みは`switchMap`など）は、`rxMethod`でもそのまま当てはまります。

`rxMethod`で定義したメソッドは、コンポーネントから呼べます。

```ts
this.store.loadTasks();
```

`rxMethod`の面白いところは、値を1回渡して呼ぶだけでなく、SignalやObservableを渡せる点です。渡した入力が変わるたびに、ストリームが自動で再実行されます。たとえば、検索キーワードのSignalを渡しておけば、入力が変わるたびに検索が走る、という書き方ができます。

## tapResponseでエラーを安全に扱う

`rxMethod`の中でも、Effectのときと同じく、エラーの扱いが重要です。ストリームにエラーが伝わると、そのメソッドは以降動かなくなってしまうからです。

ここで、Effectのエラー処理の章で触れた`@ngrx/operators`の`tapResponse`が活躍します。成功時と失敗時のハンドラを分けて書け、内部でエラーを捕まえてくれるので、ストリームが止まりません。

```ts
tapResponse({
  next: (tasks) => patchState(store, { tasks, loading: false }),
  error: (error: Error) => patchState(store, { loading: false, error: error.message }),
});
```

`next`に成功時の処理、`error`に失敗時の処理を書きます。両方を必ず書かせる形になっているので、エラー処理の書き忘れを防げます。Effectでは`catchError`をInnerに置くのが定石でしたが、SignalStoreの`rxMethod`では、この`tapResponse`が同じ役割を、より安全な形で果たします。

## withEntitiesでコレクションを扱う

Classic Storeで`@ngrx/entity`を使ったように、SignalStoreにも、コレクションを正規化して管理する仕組みがあります。`@ngrx/signals/entities`の`withEntities`です。

```ts:src/app/tasks/tasks.store.ts
import { signalStore, withMethods, patchState } from '@ngrx/signals';
import { withEntities, setAllEntities, addEntity, updateEntity, removeEntity } from '@ngrx/signals/entities';

export const TasksStore = signalStore(
  { providedIn: 'root' },
  withEntities<Task>(),
  withMethods((store) => ({
    setTasks(tasks: Task[]) {
      patchState(store, setAllEntities(tasks));
    },
    add(task: Task) {
      patchState(store, addEntity(task));
    },
    toggle(id: string, completed: boolean) {
      patchState(store, updateEntity({ id, changes: { completed } }));
    },
    remove(id: string) {
      patchState(store, removeEntity(id));
    },
  })),
);
```

`withEntities<Task>()`を加えると、`entities`（全要素の配列）や`ids`といったSignalが手に入ります。更新は、`setAllEntities`や`addEntity`といった更新関数を、`patchState`に渡す形で行います。`@ngrx/entity`のAdapterとは書き方が少し違いますが、「正規化されたコレクションを、専用の関数で更新する」という考え方は同じです。コンポーネントからは、`store.entities()`でコレクションを読めます。

## withHooksでライフサイクルを扱う

Storeが初期化されたときや、破棄されるときに、処理を差し込みたいことがあります。たとえば「Storeができたら、すぐにデータを読み込む」といった場合です。これには`withHooks`を使います。

```ts:src/app/tasks/tasks.store.ts
import { withHooks } from '@ngrx/signals';

export const TasksStore = signalStore(
  { providedIn: 'root' },
  withState(/* ... */),
  withMethods(/* ... loadTasks ... */),
  withHooks({
    onInit(store) {
      store.loadTasks(); // Storeの初期化時に読み込む
    },
  }),
);
```

`onInit`はStoreが作られたとき、`onDestroy`は破棄されるときに呼ばれます。以前は、コンポーネントの`ngOnInit`で読み込みを呼んでいましたが、`withHooks`を使えば、Store側で初期化時の処理をまとめられます。コンポーネント専用のStoreなら、コンポーネントの破棄と連動して、後始末も自動で行えます。

## signalStoreFeatureで機能を再利用する

同じような状態や振る舞いを、複数のStoreで使い回したいことがあります。そのための仕組みが`signalStoreFeature`です。いくつかのフィーチャをひとまとめにして、再利用できる部品を作れます。

たとえば、Loading・Errorの章で見たcall stateを、共通のフィーチャとして切り出せます。

```ts:src/app/shared/with-call-state.ts
import { signalStoreFeature, withState, withComputed } from '@ngrx/signals';
import { computed } from '@angular/core';

export function withCallState() {
  return signalStoreFeature(
    withState<{ callState: 'idle' | 'loading' | 'loaded' | { error: string } }>({
      callState: 'idle',
    }),
    withComputed(({ callState }) => ({
      loading: computed(() => callState() === 'loading'),
    })),
  );
}
```

作った`withCallState`は、`signalStore`に並べるだけで使えます。

```ts
export const TasksStore = signalStore(
  withState(/* ... */),
  withCallState(), // 共通のcall stateを取り込む
  withMethods(/* ... */),
);
```

Classic StoreにはMeta-Reducerや`extraSelectors`という再利用の仕組みがありましたが、SignalStoreでは、この`signalStoreFeature`が、状態と振る舞いをまとめて再利用する手段になります。共通の関心事を1か所に切り出せるので、Storeが増えても重複を抑えられます。

## Events Plugin

SignalStoreが大きくなってくると、「Classic StoreのようにActionを起点にした流れがほしい」と感じることがあります。そうした要望に応えるのが、`@ngrx/signals/events`という拡張です。

これは、SignalStoreにイベント（Actionによく似た仕組み）を導入するものです。イベントを定義し、それに反応して状態を更新する、という流れを作れます。Classic Storeの持つ予測可能性と、SignalStoreの持つ簡潔さの、中間を狙った仕組みだと考えてください。比較的新しい機能なので、実際に採用するときは、使うバージョンでの位置付けを公式ドキュメントで確認することをおすすめします。ここでは、「SignalStoreにも、イベント駆動という選択肢がある」と知っておけば十分です。

## まとめ

この章では、SignalStoreの非同期処理と拡張を確認しました。

- `rxMethod`で、RxJSのストリームをStoreのメソッドにし、非同期処理を書きます。
- `tapResponse`で成功・失敗を分けて処理し、エラーでストリームを止めないようにします。
- `withEntities`と更新関数で、コレクションを正規化して管理します。
- `withHooks`で、Storeの初期化・破棄時の処理を差し込めます。
- `signalStoreFeature`で、状態と振る舞いをまとめて再利用できます。
- `@ngrx/signals/events`は、SignalStoreにイベント駆動を導入する拡張です。

次章では、局所的な状態管理であるComponentStoreと、SignalStoreへの移行を扱います。
