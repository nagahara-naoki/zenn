---
title: "Reducerとイミュータブルな状態更新"
---

前章で、状態を変えるきっかけとなるActionを設計しました。この章では、そのActionを受けて、実際に状態を更新するReducerを扱います。

Reducerは、単方向データフローの中で、唯一状態を更新できる場所です。ここが純粋関数として正しく書けているかどうかが、状態管理全体の予測可能性を決めます。逆にいえば、Reducerさえきちんと書けていれば、「状態がおかしくなったら、まずReducerを見る」と当たりをつけられます。

この章では、`createReducer`と`on`の書き方を確認したあと、NgRxで欠かせない「イミュータブルな更新」を、配列やネストした状態も含めて、パターンとして身につけます。

## Reducerは純粋関数

Reducerは、いまの状態とActionを受け取り、新しい状態を返す関数です。

```ts
(state, action) => newState;
```

このReducerには、守るべき性質が2つあります。「純粋関数であること」と「状態をイミュータブルに更新すること」です。順番に見ていきます。

まず、純粋関数についてです。純粋関数とは、同じ入力からは常に同じ出力を返し、外の世界に影響を与えない関数のことです。つまりReducerの中では、サーバーと通信したり、乱数や現在時刻を使ったり、外部の変数を書き換えたりしてはいけません。こうした「外部とのやり取り」を副作用と呼びますが、副作用はすべてEffectsの担当です。Reducerは、渡された状態とActionだけを見て、次の状態を計算します。この純粋さのおかげで、Reducerは動きが読みやすく、テストも簡単になります（テストのしやすさは、テストの章で実感できます）。

## createReducerとonで書く

Reducerは`createReducer`で作ります。最初に初期状態を渡し、続けて`on`で「どのActionが来たら、どう更新するか」を1つずつ並べます。

```ts:src/app/tasks/tasks.reducer.ts
import { createReducer, on } from '@ngrx/store';
import { tasksActions } from './tasks.actions';

const initialState: TasksState = {
  tasks: [],
  filter: 'all',
  selectedTaskId: null,
  loading: false,
  error: null,
};

export const tasksReducer = createReducer(
  initialState,
  on(tasksActions.setFilter, (state, { filter }) => ({
    ...state,
    filter,
  })),
);
```

`on`の第1引数が、反応したいActionです。第2引数が、そのActionを受けたときの処理で、いまの状態とActionのペイロードを受け取り、新しい状態を返します。この例では、`setFilter`が来たら、`filter`だけを新しい値に差し替えた状態を返しています。`{ ...state, filter }`が、まさにその「一部だけ差し替え」です。次の節で詳しく見ます。

## イミュータブルに更新する

Reducerの要が、イミュータブルな更新です。これは、いまの状態を直接書き換えず、新しいオブジェクトを作って返す、という原則です。

言葉だけだと分かりにくいので、良くない例と正しい例を並べます。

```ts
// 避けたい: 状態を直接書き換えている
on(tasksActions.setFilter, (state, { filter }) => {
  state.filter = filter; // 元のstateを直接書き換えてしまっている
  return state;
});

// 正しい: 新しいオブジェクトを作って返している
on(tasksActions.setFilter, (state, { filter }) => ({
  ...state,
  filter,
}));
```

なぜ直接書き換えてはいけないのでしょうか。NgRxやAngularは、状態が変わったかどうかを「オブジェクトが別物になったか（参照が変わったか）」で判断しています。中身を直接書き換えると、オブジェクトは同じままなので、「変わっていない」と誤判定され、画面が更新されなかったり、Selectorのメモ化が壊れたりします。

正しい書き方では、`...state`で元の状態の中身を新しいオブジェクトに展開し、変えたいところ（`filter`）だけを上書きします。こうすると、元の状態はそのまま残り、別の新しいオブジェクトが返ります。前章のセットアップで有効にしたRuntime Checksは、うっかり直接書き換えたときに警告を出してくれる安全網です。

## 配列をイミュータブルに更新する

タスクの一覧のような配列も、元の配列を変えずに、新しい配列を作って更新します。パターンは、追加・削除・更新の3つを覚えれば十分です。

**追加**は、スプレッド構文で、既存の要素に新しい要素を加えた配列を作ります。

```ts
on(tasksActions.addTask, (state, { title }) => ({
  ...state,
  tasks: [
    ...state.tasks, // 既存のタスクをすべて展開し
    { id: crypto.randomUUID(), title, completed: false }, // 末尾に新しいタスクを足す
  ],
}));
```

**削除**は、`filter`で、対象を除いた新しい配列を作ります。

```ts
on(tasksActions.removeTask, (state, { id }) => ({
  ...state,
  tasks: state.tasks.filter((task) => task.id !== id), // id が一致しないものだけ残す
}));
```

**更新**は、`map`で、対象だけを新しいオブジェクトに差し替えます。

```ts
on(tasksActions.toggleTask, (state, { id }) => ({
  ...state,
  tasks: state.tasks.map((task) =>
    task.id === id ? { ...task, completed: !task.completed } : task,
  ),
}));
```

`map`の中では、IDが一致するタスクだけ`{ ...task, completed: !task.completed }`と新しいオブジェクトにし、それ以外はそのまま返しています。

ここで避けたいのは、`push`や`splice`のような、元の配列を書き換えてしまうメソッドです。かわりに、`filter`・`map`・スプレッドといった、必ず新しい配列を返す操作を使います。これらはすべて元の配列に手を触れないので、イミュータブルの原則を自然に守れます。

## ネストした状態の更新

状態が入れ子になっていると、更新のたびに、外側の階層まで作り直す必要が出てきます。

```ts
// ネストが深いと、各階層をスプレッドで作り直すことになる
on(someAction, (state) => ({
  ...state,
  outer: {
    ...state.outer,
    inner: {
      ...state.outer.inner,
      value: newValue,
    },
  },
}));
```

いちばん奥の`value`を1つ変えるだけなのに、`inner`も`outer`も、外側をすべて新しく作り直しています。これは、変えた部分の外側も新しいオブジェクトにしなければ、参照が変わったと判断されないためです。

このわずらわしさこそ、前章で「状態をフラットに保つ」と言った理由です。状態が浅ければ、Reducerの更新も浅く、読みやすく保てます。もしこうした深い作り直しがコードに増えてきたら、それは状態設計を見直す合図だと考えてください。

## 複数のActionを1つの処理でまとめる

同じ更新を、複数のActionで行いたいこともあります。その場合は、`on`に複数のActionをまとめて渡せます。

```ts
on(
  tasksActions.loadTasks,
  tasksActions.reloadTasks,
  (state) => ({ ...state, loading: true, error: null }),
);
```

「最初の読み込み」でも「再読み込み」でも、同じように`loading`を立てたい、というときに、処理を1つにまとめられます。

## 通信の状態を更新する

前章で作った、通信の3つのAction（要求・成功・失敗）に対応するReducerを書いてみます。ここまでのパターンの組み合わせです。

```ts
export const tasksReducer = createReducer(
  initialState,
  on(tasksActions.loadTasks, (state) => ({
    ...state,
    loading: true, // 読み込みを始めたので、ローディングを立てる
    error: null, //  前回のエラーは消す
  })),
  on(tasksActions.loadTasksSuccess, (state, { tasks }) => ({
    ...state,
    tasks, //         返ってきた一覧で差し替える
    loading: false, // ローディングを下ろす
  })),
  on(tasksActions.loadTasksFailure, (state, { error }) => ({
    ...state,
    loading: false, // ローディングを下ろす
    error, //          エラーを記録する
  })),
);
```

要求で`loading`を立て、成功で一覧を入れて`loading`を下ろし、失敗で`error`を入れて`loading`を下ろす。通信の状態が、Actionの流れにそって素直に移り変わっていくのが読み取れます。ここで見てほしいのは、実際の通信そのものがReducerに一切書かれていないことです。通信はEffectsが担当し、Reducerはその「結果の出来事」を受けて、状態に反映するだけです。役割分担が、コードにそのまま表れています。

## まとめ

この章では、Reducerとイミュータブルな更新を確認しました。

- Reducerは純粋関数です。副作用は書かず、Effectsに任せます。
- `createReducer`と`on`で、Actionごとの更新処理を並べます。
- 状態は直接書き換えず、スプレッド構文で新しいオブジェクトを作って返します。
- 直接書き換えると、変更が検知されず、画面更新やメモ化が壊れます。
- 配列は`filter`・`map`・スプレッドで、新しい配列を作って更新します。
- ネストが深いと更新が複雑になり、フラットな状態設計の大切さが効いてきます。

次章では、増えていくReducerとActionを、機能ごとにまとめる`createFeature`を扱います。
