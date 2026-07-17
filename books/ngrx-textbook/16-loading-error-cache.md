---
title: "Loading・Error・Cacheを設計する"
---

前章までで、状態の読み書きと副作用が、ひととおりつながりました。この章では、実務でほぼ必ず登場する、通信にまつわる状態の設計を扱います。読み込み中の表示、エラーの扱い、そしてキャッシュです。

これらは、素朴に書くと、フラグがあちこちに散らばり、状態が絡まりやすい部分です。「読み込み中フラグ」「エラーフラグ」「データがあるかフラグ」と、booleanが増えていくと、その組み合わせを正しく保つのが難しくなります。整理された設計の型を知っておくと、通信状態を一貫して、混乱なく扱えます。

タスク管理アプリの読み込みを題材に、状態の持ち方を組み立てていきます。

## booleanフラグの限界

まず、素朴なやり方から始めます。読み込み状態を`loading`というbooleanで、エラーを`error`という文字列で持つ、という形です。

```ts
type TasksState = {
  tasks: Task[];
  loading: boolean;
  error: string | null;
};
```

状態が小さいうちは、これで十分です。しかし、これには落とし穴があります。booleanを組み合わせると、本来あり得ない状態まで表現できてしまうのです。

たとえば、`loading`が`true`で、なおかつ`error`にも値が入っている状態は、あり得るのでしょうか。「読み込み中なのに、エラーも起きている」というのは、矛盾しています。しかし、この型では、その矛盾した状態を作れてしまいます。ほかにも、「まだ一度も読み込んでいない状態」と「読み込んだ結果、たまたま0件だった状態」を、`tasks: []`だけでは区別できません。booleanの組み合わせでは、こうした違いがあいまいになります。

## call stateパターン

そこで、通信の状態を「1つの値」で表す方法があります。取り得る状態を、文字列などの集合として、はっきり定義するのです。

```ts
type CallState = 'idle' | 'loading' | 'loaded' | { error: string };

type TasksState = {
  tasks: Task[];
  callState: CallState;
};
```

`callState`は、この4つのうち、必ずどれか1つだけを取ります。`idle`はまだ読み込んでいない、`loading`は読み込み中、`loaded`は成功して読み込み済み、`{ error }`は失敗、を表します。

この形の良いところは、状態が排他的になることです。`callState`は同時に2つの値を取れないので、「読み込み中かつエラー」という矛盾した状態は、そもそも表現できません。型を見れば、通信が取り得る状態が4つだと、ひと目でわかります。booleanの組み合わせで悩んでいたことが、すっきり解消します。

Reducerは、Actionにそって`callState`を移していきます。

```ts
on(tasksActions.loadTasks, (state) => ({ ...state, callState: 'loading' })),
on(tasksActions.loadTasksSuccess, (state, { tasks }) => ({
  ...state,
  tasks,
  callState: 'loaded',
})),
on(tasksActions.loadTasksFailure, (state, { error }) => ({
  ...state,
  callState: { error },
})),
```

要求で`loading`へ、成功で`loaded`へ、失敗で`{ error }`へ。状態が、はっきりと1つずつ遷移していきます。

## 状態から表示用の値を導く

`callState`という1つの値で状態を持つと、画面側は少し使いにくいと感じるかもしれません。テンプレートで「読み込み中か」を判定するのに、毎回`callState === 'loading'`と書くのは面倒です。

そこで、State設計とSelectorの章で学んだ「派生値はSelectorで導く」原則を使います。`callState`から、画面が使いやすい値を計算するSelectorを用意します。

```ts
export const selectLoading = createSelector(
  tasksFeature.selectCallState,
  (callState) => callState === 'loading',
);

export const selectError = createSelector(
  tasksFeature.selectCallState,
  (callState) =>
    typeof callState === 'object' ? callState.error : null,
);
```

これで、コンポーネントは`selectLoading`や`selectError`を読むだけで済みます。状態の内部表現は`callState`という1つの値でも、画面は`loading`や`error`という使いやすい形で受け取れます。内部の持ち方（矛盾のない1つの値）と、外への見せ方（使いやすい個別の値）を、Selectorが仲立ちしてくれるわけです。

## キャッシュ — 二重取得を避ける

次はキャッシュです。同じデータを何度も取りにいくのは、無駄な通信です。すでに読み込み済みなら、再取得しないようにしたい、という要望はよくあります。

ここで、Effectのエラー・状態参照の章で学んだ`concatLatestFrom`が生きてきます。Effectの中で現在の状態を参照し、読み込み済みなら通信をやめます。

```ts
export const loadTasks = createEffect(
  (actions$ = inject(Actions), store = inject(Store), api = inject(TaskApi)) =>
    actions$.pipe(
      ofType(tasksActions.loadTasks),
      concatLatestFrom(() => store.select(tasksFeature.selectCallState)),
      filter(([, callState]) => callState !== 'loaded'), // 読み込み済みなら止める
      switchMap(() =>
        api.getTasks().pipe(
          map((tasks) => tasksActions.loadTasksSuccess({ tasks })),
          catchError((error) =>
            of(tasksActions.loadTasksFailure({ error: error.message })),
          ),
        ),
      ),
    ),
  { functional: true },
);
```

`filter(([, callState]) => callState !== 'loaded')`が、キャッシュの判定です。`callState`がすでに`'loaded'`なら、そこで流れを止め、通信には進みません。これで、画面を開くたびに`loadTasks`をdispatchしても、一度読み込めば、二度目からは通信が飛ばなくなります。取得済みのデータをStoreが覚えているので、それをそのまま使います。

## キャッシュの無効化

キャッシュには、いつか古くなる、という問題がついて回ります。ずっと使い回していると、サーバー側でデータが変わっても、それが画面に反映されません。この「古くなったキャッシュをどう新しくするか」を、キャッシュの無効化と呼びます。

対処の方針は、いくつかあります。

- **明示的な再取得**: 更新ボタンなどで、キャッシュを無視して取り直すActionを別に用意する
- **書き込み後の反映**: 追加・更新・削除が成功したら、その結果を状態に反映する
- **有効期限**: データを取得した時刻を状態に持ち、一定時間を過ぎたら取り直す

タスクの追加や削除では、通信の成功Actionで、前章のAdapterを使って状態を直接更新するのが素直です。わざわざ全件を取り直さなくても、変更のあった分だけを反映すれば、表示と状態は一致します。有効期限による無効化は、実装が重くなりがちなので、本当に必要な箇所に絞って使うのがよいでしょう。

## 通信状態はFeatureごとに持つ

最後に、通信状態をどこに持つかです。読み込み中やエラーは、Featureごとに持ちます。もしアプリ全体で1つの`loading`を共有してしまうと、それがどの通信の状態なのか分からなくなります。

たとえば、タスクの読み込みと、ユーザー情報の読み込みは、別々の`callState`で管理します。こうすれば、1つの画面が複数の通信を同時に行っても、それぞれの状態を独立して表示できます。「タスクは読み込み中だが、ユーザー情報は表示済み」といった状況も、正しく扱えます。共通化したくなったときは、`callState`のような型と、そこから値を導くSelectorを、機能をまたいで再利用する形にとどめます。この再利用の仕組みは、SignalStoreの拡張を扱う章でも、別の形で登場します。

## まとめ

この章では、通信にまつわる状態の設計を確認しました。

- booleanフラグは、状態が増えると、あり得ない組み合わせまで表現できてしまいます。
- `idle`・`loading`・`loaded`・`error`を1つの値で表すcall stateパターンで、状態を排他的にできます。
- 表示用の`loading`や`error`は、Selectorで`callState`から導きます。
- `concatLatestFrom`で状態を参照し、読み込み済みなら再取得を止めます。
- キャッシュは、明示的な再取得・書き込み後の反映・有効期限で無効化します。
- 通信状態はFeatureごとに持ち、どの通信の状態かを明確に保ちます。

次章では、これまでの部品を、大規模なアプリケーションでどう構成するかを扱います。Featureの分割と、遅延読み込みへの対応を見ます。
