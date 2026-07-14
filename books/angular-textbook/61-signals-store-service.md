---
title: "第50章 SignalsによるStore Service"
---

前章で、BehaviorSubjectによるStore Serviceを学びました。この章では、それをSignalで書き換えます。第6部で学んだSignalは、「現在の値を持ち、変化を伝える」という点で、状態管理と相性が抜群です。BehaviorSubjectが担ってきた役割の多くを、Signalはより簡潔に果たせます。

モダンAngularでは、共有状態の管理も、Signalで書くのが主流になりつつあります。購読の管理が不要になり、テンプレートでの扱いも単純になります。この章では、前章と同じカートの例をSignalで書き換え、両者を比較しながら、Signalベースの状態管理の利点を確かめます。あわせて、この延長線上にある、NgRxのSignalStoreへの橋渡しにも触れます。

:::message
**この章で学ぶこと**

- SignalによるStore Serviceの実装
- `computed()`による派生状態
- BehaviorSubject版との比較
- Signalベースの状態管理の利点
:::

## Signalで状態を保持する

前章のカートを、Signalで書き換えます。状態は`signal()`で持ち、外へは読み取り専用の形で公開します。

```ts:src/app/cart.ts
import { computed, Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class CartService {
  // 状態を保持する（書き換え可能なのはService内だけ）
  private readonly itemsState = signal<CartItem[]>([]);

  // 読み取り専用で公開する
  readonly items = this.itemsState.asReadonly();

  add(item: CartItem): void {
    this.itemsState.update((items) => [...items, item]);
  }

  remove(id: string): void {
    this.itemsState.update((items) => items.filter((i) => i.id !== id));
  }
}
```

`itemsState`を`private`にして、書き換えられるのをService内に限定します。外へは`asReadonly()`で読み取り専用のSignalとして公開します。前章の`asObservable()`に対応する考え方です。更新は`update()`で、新しい配列に差し替えます。第27章で学んだ、不変な更新の作法がここでも活きます。

## computed()で派生状態を作る

合計金額や商品数といった派生状態は、`computed()`で定義します。前章では`map`を使いましたが、Signalでは`computed()`がその役割を担います。

```ts:src/app/cart.ts
readonly totalCount = computed(() =>
  this.items().reduce((sum, i) => sum + i.quantity, 0),
);

readonly totalPrice = computed(() =>
  this.items().reduce((sum, i) => sum + i.price * i.quantity, 0),
);
```

`items()`が変わるたびに、`totalCount`と`totalPrice`が自動で再計算されます。第29章で学んだ`computed()`の遅延評価とメモ化により、無駄な計算も避けられます。`items$.pipe(map(...))`と比べると、`computed()`のほうが直感的で、パイプの記述も要りません。

## Componentから使う

Componentでの利用は、さらに簡潔になります。`async`パイプが不要になるためです。

```ts:src/app/cart-view.ts
@Component({
  selector: 'app-cart-view',
  template: `
    <p>合計: {{ cart.totalPrice() }}円</p>
    @for (item of cart.items(); track item.id) {
      <div>
        {{ item.name }}
        <button (click)="cart.remove(item.id)">削除</button>
      </div>
    }
  `,
})
export class CartView {
  protected readonly cart = inject(CartService);
}
```

テンプレートでは、`cart.totalPrice()`や`cart.items()`と、Signalをそのまま呼び出すだけです。`async`パイプも、購読の管理も要りません。ローカルの中間プロパティを用意する必要すらなく、Serviceを直接テンプレートから使えます。コードが目に見えて減っているのがわかります。

## BehaviorSubject版との比較

前章のBehaviorSubject版と、本章のSignal版を比べます。

| 観点 | BehaviorSubject版 | Signal版 |
|---|---|---|
| 状態の保持 | `new BehaviorSubject(初期値)` | `signal(初期値)` |
| 読み取り公開 | `asObservable()` | `asReadonly()` |
| 派生状態 | `pipe(map(...))` | `computed(...)` |
| テンプレート | `items$ \| async` | `items()` |
| 購読の管理 | `async`パイプが担う | 不要 |

構造は対応していますが、Signal版のほうが、記述が簡潔で、購読の概念が表に出てきません。とくにテンプレートでの扱いやすさは、大きな違いです。単純な共有状態の管理であれば、現在はSignalベースが第一の選択肢になります。

ただし、非同期の複雑な流れ（`switchMap`による連鎖など）が絡む場合は、RxJSの出番です。その場合は、第41章で学んだ`toSignal()`などで橋渡しし、状態の保持はSignal、非同期の制御はRxJS、と役割分担します。

## NgRx SignalStoreへの橋渡し

このSignalベースのStore Serviceを、さらに構造化し、機能を追加していくと、状態管理ライブラリの領域に近づきます。実は、この部の後半で扱うNgRxのSignalStoreは、まさにこの「SignalによるStore Service」を、より体系的に、再利用しやすくしたものです。

自前のSignal Store Serviceは、小〜中規模の共有状態には十分です。しかし、多数の状態、複雑な派生、共通のパターンの再利用が必要になると、自前の実装では限界が見えてきます。そこで、SignalStoreのような専用の仕組みが選択肢になります。この章で書いた自前のStoreは、そうしたライブラリが何を自動化してくれるのかを理解するための、よい出発点です。まず自分で書いてみることで、ライブラリの価値がわかります。

## 非同期とローディング状態を扱う

サーバーからのデータ取得を、Signal Storeに取り込む例も見ておきましょう。読み込み中やエラーの状態も、Signalで一緒に持ちます。

```ts:src/app/product-store.ts
@Injectable({ providedIn: 'root' })
export class ProductStore {
  private readonly service = inject(ProductService);

  private readonly itemsState = signal<Product[]>([]);
  private readonly loadingState = signal(false);

  readonly items = this.itemsState.asReadonly();
  readonly loading = this.loadingState.asReadonly();

  async load(): Promise<void> {
    this.loadingState.set(true);
    const products = await firstValueFrom(this.service.getProducts());
    this.itemsState.set(products);
    this.loadingState.set(false);
  }
}
```

`items`と`loading`を、それぞれSignalで持ちます。Componentは、`store.loading()`が`true`のあいだローディングを表示し、`store.items()`でデータを表示します。第9部の`httpResource()`が、こうした「データ・読み込み中・エラー」の管理を自動化してくれたことを思い出すと、Storeが担う仕事の一部が見えてきます。単純な取得なら`httpResource()`を、状態を集約して共有したいならこうしたStoreを、と使い分けます。取得したデータを、複数の画面で共有し、さらに加工や更新も行うなら、Storeにまとめる価値があります。逆に、ある画面が表示するだけのデータなら、`httpResource()`で十分なことが多いでしょう。

## よくあるつまずき

- **公開する状態を書き換え可能にする**: `signal()`をそのまま公開すると、外から`set`できてしまいます。`asReadonly()`で読み取り専用にし、変更はメソッド経由に限定します。
- **`computed()`で済む派生を状態として持つ**: 合計金額のような派生値を、別のSignalとして持って手動更新すると、整合性が崩れます。派生は`computed()`で定義し、自動で追従させます。
- **オブジェクトの中身を書き換える**: `update()`を使わずに状態オブジェクトの中身を直接変えると、変更が伝わらないことがあります。新しい値に差し替えます。
- **RxJSが必要な処理まで無理にSignalで書く**: `debounceTime`や`switchMap`が要る非同期は、RxJSを使い、`toSignal()`で橋渡しします。Signalと`effect()`だけで複雑な非同期を組もうとすると、かえって複雑になります。
- **状態を細切れのSignalに分けすぎる**: 関連する状態は、ひとつのオブジェクトのSignalにまとめるか、意味のある単位で分けます。無闇に多数のSignalに分割すると、整合性のある更新が難しくなります。

## まとめ

- SignalによるStore Serviceは、`signal()`で状態を持ち、`asReadonly()`で公開します
- 更新は`update()`で不変に行い、派生状態は`computed()`で定義します
- `async`パイプや購読の管理が不要になり、BehaviorSubject版より簡潔です
- 単純な共有状態の管理は、現在Signalベースが第一の選択肢です
- この延長線上に、後半で学ぶNgRx SignalStoreがあります

次章では、より大規模な状態管理の思想である、NgRxとReduxパターンを学びます。
