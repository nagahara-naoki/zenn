---
title: "ComponentStoreとSignalStoreへの移行"
---

SignalStoreの締めくくりとして、ComponentStoreを扱います。ComponentStoreは、SignalStoreより前から使われてきた、局所的な状態管理の仕組みです。

「これから学ぶなら、なぜ古いほうを?」と思うかもしれません。理由は、既存のプロジェクトには、ComponentStoreで書かれたコードがまだ多く残っているからです。それを読み解き、必要ならSignalStoreへ移行できるように、両者の対応を整理しておく価値があります。

この章では、ComponentStoreの基本を確認し、SignalStoreと比べ、移行の道筋を見ていきます。

## ComponentStoreとは

ComponentStoreは、`@ngrx/component-store`が提供する、コンポーネント単位の状態管理です。

ここまで見てきたClassic Storeは、アプリ全体の状態を扱うものでした。それに対してComponentStoreは、1つのコンポーネント（とその子）に閉じた、局所的な状態を扱います。画面を開いているあいだだけ必要で、ほかの画面とは共有しない状態に向いています。

仕組みはRxJSベースで、状態をObservableとして読み出し、`updater`で更新し、`effect`で副作用を扱います。コンポーネントの`providers`で提供するので、そのコンポーネントの寿命と、状態の寿命が連動します。画面を閉じれば、状態も破棄されます。

## ComponentStoreの基本

ComponentStoreは、クラスとして定義します。`ComponentStore`を継承し、初期状態をコンストラクタで渡します。

```ts:src/app/tasks/tasks.store.ts
import { ComponentStore } from '@ngrx/component-store';
import { Injectable } from '@angular/core';
import { switchMap, tap } from 'rxjs';

@Injectable()
export class TasksStore extends ComponentStore<TasksState> {
  constructor(private readonly api: TaskApi) {
    super({ tasks: [], filter: 'all', loading: false });
  }

  // 状態の読み出し（Observable）
  readonly tasks$ = this.select((state) => state.tasks);
  readonly filter$ = this.select((state) => state.filter);

  // 更新
  readonly setFilter = this.updater((state, filter: Filter) => ({
    ...state,
    filter,
  }));

  // 副作用
  readonly loadTasks = this.effect<void>((trigger$) =>
    trigger$.pipe(
      tap(() => this.patchState({ loading: true })),
      switchMap(() =>
        this.api.getTasks().pipe(
          tapResponse({
            next: (tasks) => this.patchState({ tasks, loading: false }),
            error: (error: Error) => this.patchState({ loading: false }),
          }),
        ),
      ),
    ),
  );
}
```

3つの部品が登場します。`select`で状態を読み出し（Observableが返ります）、`updater`で同期的な更新を定義し、`effect`で非同期処理を書きます。コンポーネントの`providers`に`TasksStore`を置くと、そのコンポーネント専用のStoreになります。

## SignalStoreとの対応

このComponentStoreのコードを、前章までのSignalStoreと見比べてみましょう。どちらも局所的な状態管理を担い、構成もよく似ています。対応を表にします。

| ComponentStore | SignalStore |
|---|---|
| `ComponentStore`を継承したクラス | `signalStore`関数 |
| `select`（Observable） | 状態のSignal・`withComputed` |
| `updater` | `withMethods`＋`patchState` |
| `effect` | `rxMethod` |
| `providers`で提供 | コンポーネント提供、または`providedIn` |

見比べると、担う役割はほぼ同じで、書き方が違うだけだとわかります。いちばん大きな違いは、状態の表現です。ComponentStoreはObservableを基本にし、SignalStoreはSignalを基本にします。Angularの現行世代がSignalsを標準とすることを考えると、新しく局所状態を作るなら、SignalStoreが素直な選択です。

## ComponentStoreの位置付け

ここで、ComponentStoreの現在の立ち位置を、はっきりさせておきます。ComponentStoreは、いまも問題なく利用でき、動作します。既存のコードを、あわてて書き換える必要はありません。

一方で、NgRxは、SignalsベースのSignalStoreを、局所的な状態管理の主軸として推していく方向にあります。つまり、これから新しく局所状態を作るなら、SignalStoreを選ぶのが、今後の標準に沿った選択です。ComponentStoreは、「既存コードを読み解き、保守するための知識」として押さえておくのがよいでしょう。この関係は、Classic Storeと新しいAPIの関係にも、どこか似ています。古いものを否定するのではなく、読める状態にしておきつつ、新しいものへ移っていく、という姿勢です。

## SignalStoreへ移行する

ComponentStoreからSignalStoreへ移すときは、先ほどの対応表に沿って、1つずつ置き換えます。

`select`は、状態のSignalか`withComputed`に移します。

```ts
// ComponentStore
readonly tasks$ = this.select((state) => state.tasks);

// SignalStore（withStateで tasks は自動でSignalになるので、store.tasks() で読める）
```

`updater`は、`withMethods`のメソッドに移します。

```ts
// ComponentStore
readonly setFilter = this.updater((state, filter: Filter) => ({ ...state, filter }));

// SignalStore
withMethods((store) => ({
  setFilter(filter: Filter) {
    patchState(store, { filter });
  },
}));
```

`effect`は、`rxMethod`に移します。前章で見たとおり、内側のストリームの組み方はほとんど同じです。全体として、移行は対応表に沿った機械的な置き換えで進められる部分が多く、RxJSの知識もそのまま生きます。ゼロから書き直すというより、翻訳に近い作業です。

## 局所状態はどこで提供するか

SignalStoreを局所的に使うときは、コンポーネントの`providers`で提供します。すると、そのコンポーネントが破棄されるとき、Storeも一緒に破棄されます。

```ts
@Component({
  providers: [TasksStore], // このコンポーネント専用のStore
})
export class TaskPageComponent {
  readonly store = inject(TasksStore);
}
```

ここで、提供する場所の使い分けを整理しておきます。アプリ全体で共有したいなら`{ providedIn: 'root' }`、特定の画面に閉じたいならコンポーネントの`providers`で提供する、と選びます。局所状態をコンポーネント提供にすると、状態の寿命が画面と一致し、画面を閉じたときの後始末も自動になります。「その状態は、どこまでの範囲で必要か」を考えて、提供する場所を決めるのがコツです。

## まとめ

この章では、ComponentStoreとSignalStoreへの移行を確認しました。

- ComponentStoreは、コンポーネント単位の局所状態を扱う、RxJSベースの仕組みです。
- `select`・`updater`・`effect`で、状態の読み出し・更新・副作用を書きます。
- SignalStoreとは`select`↔Signal、`updater`↔`withMethods`、`effect`↔`rxMethod`が対応します。
- ComponentStoreは現在も使えますが、新規の局所状態にはSignalStoreが標準の方向です。
- 移行は対応表に沿った置き換えで進められ、RxJSの知識も生きます。
- 局所状態はコンポーネント提供にすると、寿命が画面と一致し、後始末も自動になります。

次章からは、テストに入ります。まず、純粋関数であるReducerとSelectorのテストから始めます。
