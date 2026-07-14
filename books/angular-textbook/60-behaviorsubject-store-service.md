---
title: "BehaviorSubjectによるStore Service"
---

前章で、共有される状態はServiceに置くと整理しました。この章では、その古典的な実装である、BehaviorSubjectを使ったStore Serviceを学びます。第40章でBehaviorSubjectの基本に触れ、第8部の終わりで少し例を見ましたが、ここでは状態管理の観点から、あらためて体系的に扱います。

BehaviorSubjectによるStore Serviceは、NgRxのような大掛かりなライブラリを導入せずに、共有状態を管理する定番の手法でした。RxJSさえあれば実装でき、多くのプロジェクトで使われてきました。モダンAngularでは、次章のSignalベースの手法が主流になりつつありますが、既存コードには本手法が数多く残っており、状態管理の考え方を学ぶうえでも重要です。まずは、この古典から押さえましょう。

:::message
**この章で学ぶこと**

- Store Serviceの基本構造
- BehaviorSubjectによる状態の保持と公開
- 状態の読み取りと更新の分離
- この手法の利点と課題
:::

## Store Serviceの考え方

Store Serviceとは、アプリケーションの共有状態を保持し、その読み取りと更新の窓口を提供するServiceのことです。「Store（貯蔵庫）」の名のとおり、状態を一か所に集めて管理します。

基本的な構造は、次の3つの要素からなります。

- **状態の保持**: 現在の状態を、Serviceの中に持つ
- **状態の公開**: 状態を、外から読み取れる形で提供する
- **状態の更新**: 状態を変更するためのメソッドを提供する

重要なのは、状態を直接書き換えさせず、必ず更新用のメソッドを通させることです。これにより、状態がいつ、どこで変わるのかを、Serviceの中に限定できます。第17章で学んだ単方向データフローの考え方が、状態管理にも貫かれます。Componentは状態を読むことと、更新を「依頼する」ことはできても、直接書き換えることはできない。この制約が、状態の変更経路を明確に保ち、「なぜこの値になったのか」を追いやすくします。制約は不自由に見えて、実は見通しのよさをもたらすのです。

## BehaviorSubjectで状態を保持する

BehaviorSubjectは、「現在の値を持つObservable」でした。この性質が、状態の保持にうってつけです。カートの状態を管理する`CartService`を例に、実装を見ていきます。

```ts:src/app/cart.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class CartService {
  // 状態を保持する（外からは触れない）
  private readonly itemsSubject = new BehaviorSubject<CartItem[]>([]);

  // 状態を読み取り専用で公開する
  readonly items$ = this.itemsSubject.asObservable();

  add(item: CartItem): void {
    const current = this.itemsSubject.value;
    this.itemsSubject.next([...current, item]);
  }

  remove(id: string): void {
    const next = this.itemsSubject.value.filter((i) => i.id !== id);
    this.itemsSubject.next(next);
  }
}
```

`itemsSubject`は`private`にして、外から直接触れないようにします。外へは`asObservable()`で読み取り専用の`items$`として公開します。第40章で触れたこの使い分けが、状態の変更経路を守る鍵です。状態を変えられるのは、`add`や`remove`といったメソッドだけになります。

## 派生した状態を作る

状態から計算して求まる値、たとえば「カートの合計金額」や「商品数」も、Observableとして公開できます。第39章で学んだ`map`を使います。

```ts:src/app/cart.ts
import { map } from 'rxjs';

// items$ から派生する状態
readonly totalCount$ = this.items$.pipe(
  map((items) => items.reduce((sum, i) => sum + i.quantity, 0)),
);

readonly totalPrice$ = this.items$.pipe(
  map((items) => items.reduce((sum, i) => sum + i.price * i.quantity, 0)),
);
```

`items$`が変わるたびに、`totalCount$`や`totalPrice$`も自動で新しい値を流します。派生した状態を、そのつど計算するのではなく、宣言的に定義できるのが、RxJSベースのStoreの特徴です。

## Componentから使う

Componentは、このServiceを注入し、公開されたObservableを`async`パイプで表示します。状態を変えたいときは、Serviceのメソッドを呼びます。

```ts:src/app/cart-view.ts
@Component({
  selector: 'app-cart-view',
  template: `
    <p>合計: {{ totalPrice$ | async }}円</p>
    @for (item of items$ | async; track item.id) {
      <div>
        {{ item.name }}
        <button (click)="cart.remove(item.id)">削除</button>
      </div>
    }
  `,
})
export class CartView {
  protected readonly cart = inject(CartService);
  protected readonly items$ = this.cart.items$;
  protected readonly totalPrice$ = this.cart.totalPrice$;
}
```

Componentは、状態を`async`パイプで表示し、`cart.remove()`で更新を依頼するだけです。状態そのものを保持したり、書き換えたりはしません。この役割分担により、複数のComponentが同じカートの状態を共有し、どこかで変更すれば、すべてに反映されます。

## この手法の利点と課題

BehaviorSubjectによるStore Serviceには、明確な利点があります。追加のライブラリが不要で、RxJSだけで実装できること。仕組みが単純で、理解しやすいこと。中小規模のアプリには、これで十分なことが多いものです。

一方、課題もあります。状態が増えると、`asObservable()`や`map`の記述が増え、定型的なコードがかさみます。また、`async`パイプでの購読が前提となり、テンプレートやロジックにObservableが多く登場します。さらに、状態が複雑に絡み合うと、更新の流れを追いにくくなることもあります。

これらの課題のうち、記述の煩雑さと購読の手間は、次章で学ぶSignalベースのStore Serviceが解消します。そして、大規模で複雑な状態の追いにくさは、NgRxのような、より構造化された仕組みが解決します。BehaviorSubjectによるStoreは、その出発点として、状態管理の基本形を教えてくれます。

## 非同期をStoreに取り込む

実際のStoreでは、サーバーからのデータ取得が絡むことがよくあります。BehaviorSubjectによるStoreでは、通信の結果を受け取って、状態を更新します。第9部で学んだHttpClientと組み合わせます。

```ts:src/app/cart.ts
private readonly service = inject(CartService);

load(): void {
  this.service.fetchItems().subscribe((items) => {
    this.itemsSubject.next(items); // 取得結果で状態を更新
  });
}
```

通信結果を`next`で状態に反映します。ここで、読み込み中やエラーの状態も、あわせて管理したくなります。そうなると、`itemsSubject`のほかに`loadingSubject`や`errorSubject`も必要になり、Subjectの数が増えていきます。この管理の煩雑さが、次章のSignalベース、さらにはNgRxのような仕組みが求められる背景のひとつです。小さなStoreでは問題になりませんが、状態が増えると、この煩雑さは無視できなくなります。

## よくあるつまずき

- **Subjectをそのまま公開する**: `itemsSubject`を`public`にすると、どこからでも`next`で状態を書き換えられ、変更経路が追えなくなります。必ず`asObservable()`で読み取り専用にして公開します。
- **状態を直接書き換える**: `this.itemsSubject.value.push(item)`のように現在の配列を直接書き換えると、変化が正しく伝わらないことがあります。新しい配列を作って`next`します。
- **購読の解除を忘れる**: Store内で他のObservableを購読する場合、その解除を忘れるとメモリリークになります。`takeUntilDestroyed()`などで対処します。

なお、`providedIn: 'root'`のStore Serviceは、アプリ全体でひとつのインスタンスが共有される（第25章）点も、あらためて意識しておきましょう。だからこそ、複数のComponentが同じ状態を見られます。逆に、Componentごとに独立した状態がほしい場合は、Componentの`providers`にStore Serviceを登録します。共有の範囲を、提供の場所で制御できるのです。この使い分けは、状態管理の設計において重要な選択肢になります。たとえば、アプリ全体で共有するカートは`providedIn: 'root'`で、特定の編集画面だけで使う一時的な状態はComponentの`providers`で、と選び分けます。状態の「寿命」と「共有範囲」を、提供の場所が決めると考えると、設計の見通しがよくなります。

## まとめ

- Store Serviceは、共有状態の保持・公開・更新の窓口を担うServiceです
- BehaviorSubjectで状態を保持し、`asObservable()`で読み取り専用に公開します
- 状態の更新はメソッドに限定し、単方向データフローを保ちます
- 派生した状態は`map`で宣言的に定義できます
- 追加ライブラリ不要で単純ですが、規模が大きくなると記述がかさむ課題があります
- 提供の場所（`root`かComponentか）で、状態の共有範囲を制御できます

次章では、この手法をSignalで書き換え、より簡潔なStore Serviceを実現する方法を学びます。
