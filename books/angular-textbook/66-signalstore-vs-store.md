---
title: "NgRx SignalStoreとNgRx Storeの使い分け"
---

第10部の締めくくりとして、NgRxが提供する2つの状態管理の仕組み、SignalStoreとStoreの使い分けを学びます。ここまで学んできたAction・Reducer・Selectorによる状態管理は、正確には「NgRx Store」と呼ばれる、伝統的な仕組みです。これに対し、NgRxには近年、Signalを土台とした「NgRx SignalStore」という新しい選択肢が加わりました。

SignalStoreは、第50章で自前で書いたSignalベースのStore Serviceを、体系的に、再利用しやすくしたものだと考えると、理解しやすくなります。Reduxパターンの厳格さよりも、簡潔さと使いやすさを重視した設計です。この章では、両者の違いと、どちらをいつ選ぶべきかを整理し、第10部全体を締めくくります。

:::message
**この章で学ぶこと**

- NgRx SignalStoreの書き方
- SignalStoreとNgRx Storeの違い
- それぞれが向く場面
- 状態管理全体のまとめ
:::

## NgRx SignalStoreの書き方

SignalStoreは、`signalStore`関数で定義します。状態・派生・メソッドを、`with〜`という部品を組み合わせて宣言します。第50章の自前Store Serviceと、同じカウンターを書いてみます。

```ts:src/app/counter.store.ts
import { signalStore, withState, withComputed, withMethods, patchState } from '@ngrx/signals';
import { computed } from '@angular/core';

export const CounterStore = signalStore(
  { providedIn: 'root' },
  withState({ count: 0 }),
  withComputed(({ count }) => ({
    doubled: computed(() => count() * 2),
  })),
  withMethods((store) => ({
    increment(): void {
      patchState(store, { count: store.count() + 1 });
    },
    add(amount: number): void {
      patchState(store, { count: store.count() + amount });
    },
  })),
);
```

要素を順に見ましょう。`withState`が状態を定義します。`withComputed`が派生状態を、`computed()`で定義します。`withMethods`が、状態を更新するメソッドを定義します。状態の更新は、`patchState`で行います。`patchState(store, { count: ... })`は、指定した部分だけを新しい値に差し替えます。

`{ providedIn: 'root' }`により、このStoreはServiceとして、アプリ全体で共有されます。Componentごとに独立させたい場合は、Componentの`providers`に登録することもできます。第25章で学んだ提供の仕組みが、そのまま使えます。

## Componentから使う

SignalStoreは、ふつうのServiceと同じように注入して使います。状態も派生も、Signalとして直接読めます。

```ts:src/app/counter.ts
@Component({
  selector: 'app-counter',
  template: `
    <p>{{ store.count() }}（2倍: {{ store.doubled() }}）</p>
    <button (click)="store.increment()">増やす</button>
  `,
})
export class Counter {
  protected readonly store = inject(CounterStore);
}
```

`store.count()`で状態を、`store.doubled()`で派生状態を、`store.increment()`で更新を行います。NgRx Storeのように、ActionをdispatchしたりSelectorを定義したりする必要はありません。第50章の自前Store Serviceと、使い勝手はほとんど同じです。それでいて、`withComputed`や、通信を扱う`rxMethod`といった部品が用意されており、機能を宣言的に組み立てられます。共通の機能は、`signalStoreFeature`で部品化し、複数のStoreで再利用することもできます。

## SignalStoreとNgRx Storeの違い

2つの仕組みの違いを、表に整理します。

| 観点 | NgRx Store | NgRx SignalStore |
|---|---|---|
| 土台 | RxJS・Reduxパターン | Signal |
| 状態変更 | Action → Reducer | `patchState` |
| 読み取り | Selector | Signalを直接読む |
| 記述量 | 多い | 少ない |
| 追跡可能性 | 高い（Action履歴） | 中程度 |
| 学習コスト | 高い | 低め |

NgRx Storeは、Action・Reducer・Selectorという厳格な構造により、変更の追跡可能性が高く、大規模で複雑な状態に強みがあります。その代わり、記述量が多く、学習コストも高めです。

SignalStoreは、Signalを直接扱う簡潔さが魅力です。記述量が少なく、Componentからの利用も単純です。Reduxほどの厳格な追跡可能性はありませんが、多くのアプリには、これで十分な構造化がもたらされます。

## それぞれが向く場面

では、どちらを選ぶべきでしょうか。判断の指針を示します。

**SignalStoreが向く場面**は、幅広くあります。中規模のアプリ、機能ごとのまとまった状態、Signalベースで一貫して書きたい場合です。記述が少なく、モダンAngularとの相性もよいため、新規開発では、まずSignalStoreを検討するのがよいでしょう。第50章の自前Store Serviceで物足りなくなったら、その自然な発展先になります。

**NgRx Store（従来のStore）が向く場面**は、より限定的です。非常に大規模で、状態の変更を厳密に追跡する必要があり、Action履歴によるデバッグや、時間を巻き戻すような高度な開発体験が重要な場合です。また、すでにNgRx Storeで書かれた大規模な既存プロジェクトも、当然その延長で保守します。

本書が推奨するのは、「まずSignalベース（自前のStore ServiceやSignalStore）から検討し、Reduxパターンの厳格な追跡可能性が本当に必要なときに、NgRx Storeを選ぶ」という方針です。かつては大規模状態管理といえばNgRx Storeが定番でしたが、Signalの登場で、選択肢が広がりました。

## 状態管理全体のまとめ

第10部を通して、状態管理の選択肢を、段階的に見てきました。ここで全体を振り返り、選択の地図を示します。

- **ローカルな状態**: ComponentのSignal（第48章）
- **共有される状態（小〜中規模）**: 自前のStore Service（Signalベース、第50章）
- **共有される状態（中〜大規模）**: NgRx SignalStore（本章）
- **大規模で厳密な追跡が必要**: NgRx Store（第51章〜第54章）

大切なのは、規模と要件に応じて選ぶことです。小さなアプリに大掛かりな仕組みを持ち込めば、複雑さだけが増します。逆に、大規模なアプリを自前のStoreで押し通せば、管理が破綻します。状態を分類し（第48章）、その性質と規模に見合った手段を選ぶ。この判断こそ、状態管理の設計の核心です。

## SignalStoreで非同期を扱う

SignalStoreにも、非同期を扱う仕組みがあります。`rxMethod`を使うと、RxJSのOperatorを活かした非同期処理を、Storeのメソッドとして組み込めます。

```ts:src/app/product.store.ts
import { signalStore, withState, withMethods, patchState } from '@ngrx/signals';
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { pipe, switchMap, tap } from 'rxjs';

export const ProductStore = signalStore(
  { providedIn: 'root' },
  withState({ products: [] as Product[], loading: false }),
  withMethods((store, service = inject(ProductService)) => ({
    load: rxMethod<void>(
      pipe(
        tap(() => patchState(store, { loading: true })),
        switchMap(() =>
          service.getProducts().pipe(
            tap((products) => patchState(store, { products, loading: false })),
          ),
        ),
      ),
    ),
  })),
);
```

`rxMethod`は、第39章で学んだ`switchMap`などのOperatorを、そのまま使えます。NgRx Storeでは、この非同期処理をEffectsという別の仕組みに切り出す必要がありましたが、SignalStoreでは、Storeの中にメソッドとして書けます。ActionもReducerも介さず、状態と非同期処理が、ひとつのStoreにまとまるのです。この簡潔さが、SignalStoreの大きな魅力です。

## よくあるつまずき

- **SignalStoreとNgRx Storeを混在させる**: ひとつのアプリで両方を使うと、状態管理の方針がぶれます。原則、どちらかに寄せます。
- **SignalStoreの状態を外から直接変える**: 状態の変更は、`withMethods`で定義したメソッドと`patchState`を通します。Storeの外から勝手に変えると、変更経路が追えなくなります。
- **不要なのにNgRx Storeを選ぶ**: 「大規模＝NgRx Store」と短絡せず、SignalStoreで足りないか、Action履歴による追跡が本当に要るかを見極めます。
- **状態管理を導入すること自体が目的になる**: 状態管理は手段です。まずComponentのSignalやServiceで足りないかを考え、必要になってから段階的に導入します。

## まとめ

- NgRx SignalStoreは、`signalStore`で状態・派生・メソッドを宣言的に組み立てます
- 状態は`patchState`で更新し、Signalとして直接読み取れます
- NgRx Storeは追跡可能性に優れ、SignalStoreは簡潔さに優れます
- **新規開発ではSignalベースを起点に、厳密な追跡が必要なときNgRx Storeを選ぶのが現在の指針です**
- 状態管理は、規模と要件に応じて手段を選ぶことがもっとも重要です

以上で第10部は終わりです。最後の第11部では、テストやパフォーマンス、セキュリティなど、実務的なAngular開発の総仕上げを学びます。
