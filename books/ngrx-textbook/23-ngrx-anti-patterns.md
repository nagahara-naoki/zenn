---
title: "NgRxのアンチパターン"
---

ここからは、本書の締めくくりです。まず、NgRxでやりがちなアンチパターンを整理します。

アンチパターンとは、一見うまく動いても、あとで問題を引き起こす書き方です。本書で学んだ原則を裏返すと、「こう書くと崩れる」という失敗の形が見えてきます。ここでは症状、問題になる理由、直し方の順に整理します。

自分の書いたコードを見直すときの、チェックリストとして使ってください。

## コマンド型のAction

最初のアンチパターンは、Actionを更新命令として設計してしまうことです。

```ts
// アンチパターン: 更新処理を名前にしている
createAction('[Tasks] Set Tasks', props<{ tasks: Task[] }>());
createAction('[Tasks] Set Loading', props<{ loading: boolean }>());
```

`Set Tasks`や`Set Loading`という名前は、「状態をどう書き換えるか」をそのまま表しています。これのどこが問題なのでしょうか。こう設計すると、1つの出来事に対して、いくつものActionを発行することになりがちです。「読み込みが成功した」という1つの出来事のために、`Set Tasks`と`Set Loading`を別々にdispatchする、といった具合です。すると、DevToolsの履歴を見ても、機械的な操作が並ぶだけで、何が起きたのかが読み取れません。

直し方は、Action設計の章で見たとおりです。Actionは出来事として設計します。「読み込みが成功した」という`loadTasksSuccess`を1つ発行し、`tasks`の更新と`loading`の解除は、それを受けたReducerがまとめて行います。1つの出来事は、1つのActionで表します。

## Storeに何でも入れる

次は、状態をStoreに入れすぎることです。

コンポーネントの中だけで完結する一時的な値まで、何でもStoreに入れてしまうと、Storeが肥大化します。フォームの入力途中の文字や、メニューが開いているかどうかといった細かいUIの状態まで持たせると、そのぶんActionも増え、アプリ全体が重く、追いにくくなります。

考え方の原則は、こうです。アプリ全体で共有し、単方向フローで厳密に管理する価値があるものだけを、Storeに置きます。コンポーネントに閉じた状態は、Signalや局所的なStoreで持てば十分なことが多いのです。何をStoreに入れ、何を入れないか。この判断は、状態管理手法の使い分けを扱う次章で、さらに詳しく考えます。

## 派生値を状態に格納する

3つ目は、計算で求められる値を、状態に持ってしまうことです。

```ts
// アンチパターン: 件数を状態に持つ
type TasksState = {
  tasks: Task[];
  incompleteCount: number; // tasksから計算できる
};
```

State設計の章で繰り返し強調したとおり、派生値を状態に持つと、元の値を更新するたびに、派生値も手で更新する必要が生じます。そして、1か所でも更新を忘れると、一覧と件数がずれます。件数や絞り込み後の一覧といった派生値は、状態に持たず、Selectorで計算します。元が変われば、Selectorが自動的に正しい値を返してくれます。この原則は、本書で最も繰り返してきた、重要なものです。

## Effect内のNested Subscribe

4つ目は、Effectの中で、購読を入れ子にしてしまうことです。

```ts
// アンチパターン: Effectの中でsubscribeしている
actions$.pipe(
  ofType(tasksActions.loadTasks),
  tap(() => {
    this.api.getTasks().subscribe((tasks) => {
      this.store.dispatch(tasksActions.loadTasksSuccess({ tasks }));
    });
  }),
);
```

購読の中でさらに購読すると、いくつも問題が起きます。キャンセルが効かず、エラー処理も難しくなり、コードも読みにくくなります。API通信の章で見たとおり、こうした場面では、`switchMap`などのFlattening Operatorで平坦化し、結果を成功・失敗のActionに変えるのが正しい書き方です。もしEffectの中で手動の`subscribe`を書いていたら、それは見直しの合図です。

## catchErrorをEffectの外に置く

5つ目は、`catchError`を、Inner Observableの外に置いてしまうことです。

Effectのエラー処理の章で詳しく見たとおり、`catchError`を`switchMap`の外に置くと、通信が一度失敗しただけで、Effect全体が止まってしまいます。以降、どんなActionが来ても反応しなくなり、「2回目以降ボタンが効かない」という分かりにくい不具合につながります。`catchError`は、API通信のInner Observableの内側に置き、失敗を失敗Actionに変えます。位置が1つずれるだけで結果が正反対になる、実務で特に多い落とし穴です。

## Selectorに重い処理や副作用を書く

6つ目は、Selectorに、副作用や重すぎる処理を書いてしまうことです。

Selectorは、状態から値を計算する純粋関数であるべきです。もしSelectorの中でAPIを呼んだり、ログを出したり、呼ばれるたびに巨大な配列を何度も走査したりすると、メモ化の利点が失われ、再描画のたびに負荷がかかります。副作用はEffectsへ、重い計算は結果を状態に持たせるなどの工夫へ、と切り分けます。Selectorは、あくまで「状態を、画面が読みたい形に変換する」ことに徹します。

## SignalStoreで状態を直接書き換える

7つ目は、SignalStoreの状態を、`patchState`を通さずに変えようとすることです。

SignalStoreの状態は、`patchState`で更新するのが基本です。Signalの`set`や`update`を使って無理に書き換えたり、状態のオブジェクトを直接いじったりすると、状態管理の一貫性が崩れます。更新は`withMethods`の中の`patchState`に集約し、更新の経路を1本に保ちます。Classic StoreでReducerに更新を集約したのと、同じ考え方です。SignalStoreでも、イミュータブルな更新の原則は変わりません。

## SignalStoreで派生値をwithStateに持つ

8つ目は、SignalStoreで、派生値を`withState`に入れてしまうことです。

```ts
// アンチパターン: 派生値をwithStateに持つ
withState({ tasks: [], visibleTasks: [] }); // visibleTasksはtasksから計算できる
```

これは、3つ目に挙げた「派生値を状態に持つ」問題の、SignalStore版です。派生値を`withState`に持つと、Classic Storeで状態に派生値を持つのと同じ矛盾が起きます。SignalStoreでは、派生値は`withComputed`で定義します。元のSignalが変わったときだけ再計算され、つねに正しい値になります。「派生値は状態に持たない」という原則は、Classic StoreでもSignalStoreでも、まったく同じように当てはまります。

## アンチパターンの共通点

ここまで並べたアンチパターンには、実は共通する根っこがあります。1つは、単方向データフローの規律を崩していること。もう1つは、状態と派生値の境界をあいまいにしていることです。

裏返せば、次の4つを守るだけで、多くのアンチパターンは自然に避けられます。

- Actionは出来事として設計する
- 状態は最小限にとどめ、派生値はSelectorや`withComputed`で計算する
- 副作用はEffectsや`rxMethod`に集約する
- 更新の経路を1本に保つ

これらは、本書を通して繰り返してきた原則そのものです。もしコードが追いにくくなってきたら、この章のパターンと照らし合わせて、どの原則が崩れているかを探してみてください。原因の見当がつくはずです。

## まとめ

この章では、NgRxのアンチパターンを確認しました。

- コマンド型のActionは避け、出来事としてActionを設計します。
- Storeに何でも入れず、共有・管理する価値があるものだけを置きます。
- 派生値は状態に持たず、Selectorや`withComputed`で計算します。
- Effect内のNested Subscribeは、Flattening Operatorで平坦化します。
- `catchError`はInner Observableの内側に置き、Effectを止めないようにします。
- SignalStoreでは`patchState`で更新し、派生値は`withComputed`に置きます。

次章では、本書の締めくくりとして、状態管理手法の使い分けと、NgRxの導入戦略を扱います。
