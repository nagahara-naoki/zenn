---
title: "付録 早見表・チェックリスト・用語集"
---

本書で扱ったAPI・設計指針・用語をまとめます。学習中や実務中の索引として使ってください。それぞれの詳しい説明は、本文の該当する章にあります。

## 付録A API早見表

### Store・Action・Reducer

| 目的 | API |
|---|---|
| Storeを登録する | `provideStore` |
| Featureの状態を登録する | `provideState` |
| Actionを定義する | `createAction` |
| Actionをまとめて定義する | `createActionGroup` |
| ペイロードの型を指定する | `props` / `emptyProps` |
| Reducerを作る | `createReducer` / `on` |
| Featureをまとめる | `createFeature` |
| Actionを発行する | `store.dispatch` |

### Selector

| 目的 | API |
|---|---|
| Selectorを作る | `createSelector` |
| 状態をObservableで読む | `store.select` |
| 状態をSignalで読む | `store.selectSignal` |
| 計算ロジックを取り出す（テスト） | `selector.projector` |

### Effects

| 目的 | API |
|---|---|
| Effectを定義する | `createEffect`（`{ functional: true }`） |
| Effectを登録する | `provideEffects` |
| Actionを絞り込む | `ofType` |
| 状態を参照する | `concatLatestFrom`（`@ngrx/operators`） |
| 成功・失敗を処理する | `tapResponse`（`@ngrx/operators`） |

### Entity

| 目的 | API |
|---|---|
| Adapterを作る | `createEntityAdapter` |
| 初期状態を作る | `adapter.getInitialState` |
| 全件を置き換える | `adapter.setAll` |
| 追加・更新・削除する | `adapter.addOne` / `updateOne` / `removeOne` |
| コレクションのSelector | `adapter.getSelectors` |

### SignalStore

| 目的 | API |
|---|---|
| Storeを作る | `signalStore` |
| 状態を定義する | `withState` |
| 派生値を定義する | `withComputed` |
| メソッドを定義する | `withMethods` |
| 状態を更新する | `patchState` |
| 非同期処理を書く | `rxMethod`（`@ngrx/signals/rxjs-interop`） |
| コレクションを扱う | `withEntities`（`@ngrx/signals/entities`） |
| ライフサイクルを扱う | `withHooks` |
| 機能を再利用する | `signalStoreFeature` |

### 連携・開発支援

| 目的 | API |
|---|---|
| ルーティング状態を取り込む | `provideRouterStore` |
| ルート情報のSelector | `getRouterSelectors` |
| デバッグする | `provideStoreDevtools` |
| Reducerを横断する処理 | Meta-Reducer（`metaReducers`） |

## 付録B 設計チェックリスト

### Stateの設計

- 派生値を状態に持っていないか（Selectorや`withComputed`で計算する）
- 選択中の項目をIDで持っているか（実体はSelectorで引く）
- コレクションは正規化を検討したか（`@ngrx/entity`）
- 状態はフラットに保たれているか
- DOM参照や一時的な入力値をStoreに入れていないか

### Actionの設計

- Actionは出来事として設計しているか（コマンドにしていないか）
- 名前は`[発生源] 出来事`の形か
- 1つの出来事に1つのActionを用意しているか
- 非同期処理は要求・成功・失敗の3つに分けたか

### Reducerの設計

- Reducerは純粋関数か（副作用を書いていないか）
- 状態をイミュータブルに更新しているか（直接書き換えていないか）
- Runtime Checksを有効にしているか

### Effectsの設計

- API通信に合ったFlattening Operatorを選んでいるか
- `catchError`をInner Observableの内側に置いているか
- Actionを流さないEffectに`{ dispatch: false }`を付けたか
- Effect内で手動`subscribe`をしていないか

### 手法の選択

- その状態は共有が必要か（不要ならSignalで足りる）
- 予測可能性を厳密に求めるか（求めるならClassic Store）
- 過剰な仕組みを導入していないか

## 付録C 用語集

**Store**
アプリケーションの状態を保持する唯一の場所。信頼できる情報源が1つだけある状態をSingle Source of Truthと呼ぶ。

**Action**
アプリケーションで「何が起きたか」を表すイベント。更新命令ではなく出来事として設計する。

**Reducer**
現在の状態とActionから新しい状態を返す純粋関数。状態を更新できる唯一の場所。

**Selector**
状態を読み出し、派生値を計算する関数。メモ化により、入力が変わらなければ再計算しない。

**Effects**
Actionを受けて副作用を実行し、新しいActionを流す仕組み。API通信などを担う。

**単方向データフロー**
データが「Action→Reducer→Store→Selector」と一方向にだけ流れる仕組み。変更を追跡でき、予測可能になる。

**メモ化**
計算結果を覚えておき、入力が変わらなければ前回の結果を再利用する仕組み。Selectorが持つ。

**Entity**
同じ型のデータが並ぶコレクションを、正規化して管理する仕組み。`@ngrx/entity`が支援する。

**正規化**
データを入れ子にせず、IDをキーにした平らな形で持つこと。重複を防ぎ、速く引ける。

**Meta-Reducer**
Reducerを包み、すべてのActionに横断的な処理を差し込む仕組み。ログや状態のリセットに使う。

**SignalStore**
Signalsをベースにした、記述の少ない状態管理。`@ngrx/signals`が提供する。

**ComponentStore**
コンポーネント単位の局所的な状態管理。RxJSベースで、SignalStoreの前から使われてきた。

**call state**
`idle`・`loading`・`loaded`・`error`のように、通信の状態を1つの排他的な値で表す設計。

**ViewModel**
画面が必要とする値を1つにまとめたSelector。コンポーネントを薄く保つ。

以上が、本書で扱った主なAPI・指針・用語です。設計に迷ったときの出発点として、本文へ戻る手がかりに使ってください。NgRxの学習が、実務での確かな判断につながることを願っています。
