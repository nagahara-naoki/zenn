---
title: "Action・Reducer・Selectorの設計"
---

前章で、NgRxの土台にあるReduxパターンの思想を学びました。この章では、その具体的な実装、すなわちAction・Reducer・Selectorの書き方を見ていきます。これらは、NgRx Storeの中核をなす3つの要素です。

Actionは「何が起きたか」を、Reducerは「それを受けて状態をどう変えるか」を、Selectorは「状態からどう読み取るか」を、それぞれ担います。この3つの役割分担が、Reduxパターンの規律を形にします。コード量は増えますが、一つひとつの要素の役割は明確です。この章では、カウンターという単純な例を通して、それぞれの書き方と、連携のしかたを学びます。

:::message
**この章で学ぶこと**

- `createAction`によるActionの定義
- `createReducer`によるReducerの実装
- `createSelector`による状態の読み取り
- Componentからの利用（dispatchとselect）
:::

## Actionを定義する

Actionは、「アプリの中で何が起きたか」を表すオブジェクトです。`createAction`で定義します。Actionには、それを識別する文字列（type）を与えます。

```ts:src/app/counter.actions.ts
import { createAction, props } from '@ngrx/store';

export const increment = createAction('[Counter] Increment');
export const decrement = createAction('[Counter] Decrement');
export const add = createAction('[Counter] Add', props<{ amount: number }>());
```

`'[Counter] Increment'`のように、`[機能名] 出来事`という形式で名付けるのが慣習です。どの機能で何が起きたかが、一目でわかります。`add`のように、追加の情報（ここでは`amount`）を伴うActionは、`props`で、その情報の型を指定します。Actionは、あくまで「起きたこと」を表すだけで、状態をどう変えるかは決めません。

## Reducerを実装する

Reducerは、Actionを受けて、現在の状態から新しい状態を計算する純粋な関数です。`createReducer`と`on`で定義します。

```ts:src/app/counter.reducer.ts
import { createReducer, on } from '@ngrx/store';
import { increment, decrement, add } from './counter.actions';

export interface CounterState {
  count: number;
}

const initialState: CounterState = { count: 0 };

export const counterReducer = createReducer(
  initialState,
  on(increment, (state) => ({ count: state.count + 1 })),
  on(decrement, (state) => ({ count: state.count - 1 })),
  on(add, (state, { amount }) => ({ count: state.count + amount })),
);
```

`createReducer`の第1引数が初期状態、続く`on`が、各Actionに対する状態の変化です。`on(increment, (state) => ...)`は、「`increment`が来たら、状態をこう変える」という定義です。ここで重要なのは、既存の状態を書き換えず、新しい状態オブジェクトを返していることです。第27章で学んだ不変性の原則が、Reducerでは必須です。Reducerは純粋な関数であり、同じ入力からは常に同じ出力を返し、副作用（通信など）を持ちません。

## Storeに登録する

定義したReducerは、アプリケーションに登録します。第7章以来の`provide`関数の流儀で、`app.config.ts`に`provideStore`を加えます。

```ts:src/app/app.config.ts
import { provideStore } from '@ngrx/store';
import { counterReducer } from './counter.reducer';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore({ counter: counterReducer }),
  ],
};
```

`provideStore({ counter: counterReducer })`で、`counter`という名前で状態を登録します。これで、アプリ全体でこの状態が使えるようになります。機能ごとに状態を分けて登録する`provideState`もあり、遅延読み込みする機能の状態は、そちらで登録します。

## Selectorで状態を読み取る

Selectorは、Storeから必要な状態を取り出す関数です。`createSelector`で定義します。派生した状態も、Selectorで計算できます。

```ts:src/app/counter.selectors.ts
import { createFeatureSelector, createSelector } from '@ngrx/store';
import { CounterState } from './counter.reducer';

// counter という状態全体を選ぶ
const selectCounter = createFeatureSelector<CounterState>('counter');

// そこから count を取り出す
export const selectCount = createSelector(selectCounter, (state) => state.count);

// 派生した状態（2倍の値）
export const selectDoubled = createSelector(selectCount, (count) => count * 2);
```

`createFeatureSelector`で状態全体を選び、`createSelector`でそこから必要な部分を取り出します。Selectorには、`createSelector`によるメモ化が働き、もとの状態が変わらなければ再計算されません。第29章の`computed()`と同じ発想です。状態の読み取りをSelectorに集約すると、状態の内部構造が変わっても、Selectorだけを直せば済みます。

## Componentから使う

Componentは、`Store`を注入し、Selectorで状態を読み取り、Actionを発行（dispatch）して状態を変更します。

```ts:src/app/counter.ts
import { Component, inject } from '@angular/core';
import { Store } from '@ngrx/store';
import { increment, add } from './counter.actions';
import { selectCount } from './counter.selectors';

@Component({
  selector: 'app-counter',
  template: `
    <p>{{ count() }}</p>
    <button (click)="store.dispatch(increment())">増やす</button>
  `,
})
export class Counter {
  protected readonly store = inject(Store);
  // Selectorの結果をSignalで受け取る
  protected readonly count = this.store.selectSignal(selectCount);
}
```

`store.selectSignal(selectCount)`で、状態をSignalとして読み取ります。かつては`store.select()`でObservableとして受け取り、`async`パイプで表示していましたが、モダンNgRxでは`selectSignal`により、Signalとして直接扱えます。第6部・第9部で見たSignalへの統一が、NgRxにも及んでいます。状態を変えるときは、`store.dispatch(increment())`のように、Actionを発行します。Component自身は状態を書き換えず、「増やす、という出来事が起きた」と伝えるだけです。

## 役割分担のまとめ

3つの要素の役割を、あらためて整理します。

| 要素 | 役割 | 例える |
|---|---|---|
| Action | 何が起きたかを表す | 「注文が入った」という伝票 |
| Reducer | 状態をどう変えるか | 伝票を見て在庫を更新する係 |
| Selector | 状態をどう読むか | 帳簿から必要な数字を読む係 |

この分業により、状態の変更は必ず「Action発行 → Reducerで計算」という経路を通り、読み取りはSelectorに集約されます。記述は増えますが、それぞれの責務が明確なため、大規模なアプリでも、変更と読み取りの流れを追いやすく保てます。これがReduxパターンの規律の、具体的な姿です。

## createFeatureでまとめる

機能ごとに、状態・Reducer・Selectorを定義していくと、関連するコードが分散しがちです。モダンNgRxには、これらをひとまとめにする`createFeature`という仕組みがあります。機能の名前・Reducerをまとめて定義すると、基本的なSelectorが自動で生成されます。

```ts:src/app/counter.feature.ts
import { createFeature, createReducer, on } from '@ngrx/store';

export const counterFeature = createFeature({
  name: 'counter',
  reducer: createReducer(
    initialState,
    on(increment, (state) => ({ count: state.count + 1 })),
  ),
});

// selectCount などが自動生成される
export const { selectCount } = counterFeature;
```

`createFeature`を使うと、`createFeatureSelector`や、状態の各プロパティに対応するSelectorを、自分で書かずに済みます。定型的なコードが減り、機能ごとのまとまりも明確になります。新しくNgRx Storeを書くなら、`createFeature`を使うのが簡潔です。

## よくあるつまずき

- **Reducerで状態を直接書き換える**: `state.count++`のように既存の状態を変更すると、変更検知や追跡が壊れます。必ず新しいオブジェクトを返します。
- **Reducerに副作用を書く**: Reducerの中で通信やログ出力をしてはいけません。純粋な関数に保ち、副作用は次章のEffectsに任せます。
- **ComponentでSelectorを使わず状態を直接読む**: 状態の読み取りは、Selectorに集約します。Componentが状態の内部構造に直接依存すると、構造変更に弱くなります。
- **Actionを状態の「命令」として書く**: Actionは「何が起きたか」という事実を表します。「countをセットせよ」ではなく「増やされた」のように、出来事として名付けます。

## まとめ

- Actionは`createAction`で定義し、「何が起きたか」を表します
- Reducerは`createReducer`と`on`で実装し、Actionを受けて新しい状態を返す純粋な関数です
- Reducerでは状態を書き換えず、不変な更新を行います
- Selectorは`createSelector`で状態を読み取り、メモ化が働きます
- Componentは`dispatch`でActionを発行し、`selectSignal`で状態をSignalとして読みます

次章では、通信などの副作用を扱うEffectsを、RxJSとともに学びます。
