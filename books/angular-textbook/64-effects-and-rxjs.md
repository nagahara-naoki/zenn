---
title: "第53章 EffectsとRxJSによる非同期処理"
---

前章で学んだReducerは、純粋な関数でした。同じ入力からは同じ出力を返し、通信のような副作用は持ちません。しかし、実際のアプリケーションでは、「保存ボタンが押されたらサーバーへ送信する」「一覧を開いたらデータを取得する」といった副作用が欠かせません。この副作用を、Reduxパターンの規律の中でどう扱うか。その答えが、NgRxのEffects（エフェクト）です。

Effectsは、Actionをきっかけに副作用を実行し、その結果をまた別のActionとして発行する仕組みです。ここで、第8部で学んだRxJSが本領を発揮します。Effectsは、Actionの流れをObservableとして受け取り、`switchMap`や`catchError`といったOperatorで、非同期処理を組み立てます。この章では、Effectsの考え方と書き方を、通信の例を通して学びます。

:::message
**この章で学ぶこと**

- Effectsが解決する課題
- `createEffect`と`ofType`によるEffectの実装
- 通信の成功・失敗をActionで表す
- Effectsの登録
:::

## なぜEffectsが必要か

Reducerは純粋な関数でなければならない、という制約がありました。この制約は、状態変更を予測可能に保つために重要です。しかし、通信のような副作用は、純粋ではありません。同じActionでも、サーバーの状態次第で結果が変わりますし、時間もかかります。これをReducerに書くことはできません。

そこで、副作用を担う専用の場所として、Effectsが用意されています。Effectsは、Storeの外側で、Actionの流れを監視します。特定のActionが発行されたら、それに反応して副作用（通信など）を実行し、その結果を新しいActionとして発行します。状態の変更（Reducer）と、副作用（Effects）を、きれいに分離するのです。

流れを整理すると、こうです。Componentが「データ取得を要求する」Actionを発行する。Effectsがそれを捉え、通信を実行する。通信が成功したら「取得成功」Action（データ入り）を、失敗したら「取得失敗」Actionを発行する。Reducerがその結果のActionを受けて、状態を更新する。副作用はEffectsに閉じ込められ、Reducerは純粋なまま保たれます。

## Effectを実装する

Effectは、`createEffect`で定義します。中身は、Actionの流れ（Observable）を受け取り、Operatorで処理する、RxJSのパイプラインです。商品一覧の取得を例に見ます。

```ts:src/app/product.effects.ts
import { inject } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { catchError, map, of, switchMap } from 'rxjs';
import { ProductService } from './product.service';
import { loadProducts, loadProductsSuccess, loadProductsFailure } from './product.actions';

export const loadProductsEffect = createEffect(
  () => {
    const actions$ = inject(Actions);
    const service = inject(ProductService);

    return actions$.pipe(
      ofType(loadProducts),                       // このActionだけに反応する
      switchMap(() =>
        service.getProducts().pipe(
          map((products) => loadProductsSuccess({ products })), // 成功Action
          catchError((error) => of(loadProductsFailure({ error }))), // 失敗Action
        ),
      ),
    );
  },
  { functional: true },
);
```

`inject(Actions)`で、アプリ全体のActionの流れを受け取ります。`ofType(loadProducts)`で、その中から`loadProducts`というActionだけを拾います。そのActionが来たら、`switchMap`で通信を実行します。第39章で学んだとおり、`switchMap`は、新しい要求が来たら前の通信を打ち切ります。通信が成功すれば`map`で成功Actionに、失敗すれば`catchError`で失敗Actionに変換します。第39章の`catchError`の使い方が、そのまま活きています。

このEffectは、`{ functional: true }`を付けた関数型で書いています。第24章・第36章で見た関数型の流れが、Effectsにも及んでいます。

## 成功と失敗をActionで表す

Effectsの設計で大切なのは、通信の結果（成功・失敗）を、必ずActionとして表すことです。先ほどの例では、`loadProductsSuccess`と`loadProductsFailure`という2つのActionを用意しました。

```ts:src/app/product.actions.ts
export const loadProducts = createAction('[Product] Load');
export const loadProductsSuccess = createAction(
  '[Product] Load Success',
  props<{ products: Product[] }>(),
);
export const loadProductsFailure = createAction(
  '[Product] Load Failure',
  props<{ error: unknown }>(),
);
```

「要求」「成功」「失敗」の3つのActionを1組にするのが、NgRxの定番のパターンです。Reducer側では、これらを受けて状態を更新します。「成功」ならデータを保存し、読み込み中フラグを下げる。「失敗」ならエラーを記録する、といった具合です。通信の各段階が、すべてActionとして記録されるため、第51章で述べた追跡可能性が保たれます。「いつデータ取得を要求し、いつ成功・失敗したか」が、Actionの履歴として残るのです。

## Effectsを登録する

作ったEffectsは、`app.config.ts`に`provideEffects`で登録します。

```ts:src/app/app.config.ts
import { provideStore } from '@ngrx/store';
import { provideEffects } from '@ngrx/effects';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore({ product: productReducer }),
    provideEffects([loadProductsEffect]),
  ],
};
```

`provideEffects`に、有効にしたいEffectを渡します。これで、対象のActionが発行されたときに、Effectが働くようになります。StoreとEffectsを両方登録することで、状態管理と副作用の仕組みがそろいます。

## Effectsを使ううえでの注意

Effectsは強力ですが、いくつか注意点があります。まず、高階Operatorの選択です。第39章で学んだとおり、`switchMap`（最新優先）・`concatMap`（順番）・`mergeMap`（並行）・`exhaustMap`（先着優先）を、処理の性質に応じて選びます。一覧取得なら`switchMap`、保存の連続なら`concatMap`、二重送信を防ぐ保存ボタンなら`exhaustMap`、といった判断が必要です。

もうひとつは、Effect内でのエラー処理です。`catchError`は、`switchMap`に渡す内側のObservableの中に置きます。もし外側の`actions$`のパイプラインでエラーを捉えてしまうと、そのEffect全体が停止し、以降のActionに反応しなくなります。エラーは内側で捉え、失敗Actionに変換して、外側の流れは止めない。これがEffectsの鉄則です。この点は、実務でつまずきやすいので、意識しておいてください。

## SignalとEffectsの関係

第29章で学んだSignalの`effect()`と、NgRxのEffectsは、名前が似ていますが、別のものです。混同しないよう整理しておきます。Signalの`effect()`は、Signalの変化に反応して副作用を実行する、Angular本体の機能でした。NgRxのEffectsは、Actionの流れに反応して副作用を実行する、NgRx固有の仕組みです。

両者は目的が異なります。Signalの`effect()`は、主にComponent内の、状態変化に応じた局所的な副作用に使います。NgRxのEffectsは、アプリ全体の状態変更（Action）に紐づく、大きな副作用（通信など）を扱います。NgRxを使う場面では、副作用はEffectsに集約するのが基本です。名前の類似に惑わされず、「Signalの変化に反応するのが`effect()`、Actionに反応するのがEffects」と区別してください。

## よくあるつまずき

- **`catchError`を外側に置く**: 前述のとおり、`catchError`を`actions$`の外側パイプラインに置くと、一度のエラーでEffect全体が止まります。内側のObservableに置きます。
- **Effect内で状態を直接書き換える**: Effectsの役割は副作用の実行と、結果のAction発行です。状態を直接変えるのではなく、結果をActionとして発行し、Reducerに任せます。
- **高階Operatorの選び違い**: 保存の連打を`switchMap`で書くと、前の保存が打ち切られてしまいます。処理の性質に応じて`concatMap`や`exhaustMap`を選びます。
- **成功Actionを発行し忘れる**: 通信しただけで結果のActionを発行しないと、状態が更新されません。成功・失敗を必ずActionで表します。

## まとめ

- Effectsは、Actionをきっかけに副作用を実行し、結果を別のActionとして発行します
- 副作用をEffectsに分離することで、Reducerを純粋なまま保てます
- `createEffect`と`ofType`で、特定のActionに反応するパイプラインを書きます
- 通信の結果は、成功・失敗のActionとして表し、追跡可能性を保ちます
- `catchError`は内側のObservableに置き、Effect全体を止めないようにします

次章では、Entity・Facade・Router Storeといった、実務的なNgRxの設計パターンを学びます。
