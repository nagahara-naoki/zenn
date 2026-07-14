---
title: "SubscriptionとObservableのライフサイクル"
---

前章で、Observableは`subscribe`されて初めて動き出すことを学びました。この章では、その`subscribe`が返すSubscription（購読）と、購読を適切に終わらせるライフサイクル管理を扱います。地味なテーマに見えますが、ここを疎かにすると、メモリリークという厄介な問題を引き起こします。RxJSを安全に使うための、重要な回です。

購読には、始まりと終わりがあります。`subscribe`で始まり、`unsubscribe`で終わります。問題は、終わらせるのを忘れると、Observableが値を流し続け、それを受け取る処理も動き続けてしまうことです。とくに、破棄されたはずのComponentに紐づく購読が生き残ると、無駄な処理やメモリの浪費につながります。この章では、その仕組みと、モダンAngularでの安全な後始末の方法を学びます。

:::message
**この章で学ぶこと**

- Subscriptionと`unsubscribe`
- 購読を解除しないと起きる問題
- Cold ObservableとHot Observable
- モダンAngularでの購読解除の方法
:::

## Subscriptionと購読解除

`subscribe`は、Subscriptionというオブジェクトを返します。これは、その購読を表す「取っ手」のようなものです。この取っ手を使って、`unsubscribe`を呼ぶと、購読を終わらせられます。

```ts
import { interval } from 'rxjs';

const subscription = interval(1000).subscribe((value) => {
  console.log(value);
});

// 購読を終わらせる
subscription.unsubscribe();
```

`unsubscribe`を呼ぶと、Observableからの値の受け取りが止まります。`interval`は、放っておくと永遠に数を流し続けるObservableですが、`unsubscribe`すれば、そこで止まります。購読を始めたら、不要になったときに終わらせる。これが、購読管理の基本です。

## 購読を解除しないと何が起きるか

購読の解除を忘れると、メモリリークが起こりえます。メモリリークとは、不要になったはずのものがメモリ上に残り続け、少しずつ資源を圧迫していく問題です。

具体的に考えてみましょう。あるComponentが、`interval`を購読して、1秒ごとに何かを処理するとします。このComponentが画面から消え、破棄されたとします。しかし、購読を解除していないと、`interval`は値を流し続け、破棄されたComponentの処理が、なおも実行され続けます。Componentは見えなくなったのに、その一部が裏で動き続けているのです。

このようなComponentが、画面遷移のたびに増えていくと、動き続ける購読がどんどん積み重なります。やがて、アプリは重くなり、不可解な動作を示すようになります。第21章で`ngOnDestroy`による後始末の重要性に触れましたが、購読の解除は、その代表例です。

## Cold ObservableとHot Observable

購読を理解するうえで、ObservableにはColdとHotの2種類がある、という区別も押さえておきましょう。

Cold Observableは、購読されるたびに、新しく値の生成を始めます。購読者ごとに、独立した流れが作られます。たとえば、HTTP通信のObservableはColdで、購読するたびに新しい通信が発生します。2人が別々に購読すれば、通信も2回起きます。水道の比喩でいえば、購読者ごとに専用の管が引かれるイメージです。

Hot Observableは、購読者の有無にかかわらず値が流れており、複数の購読者がその同じ流れを共有します。たとえば、ボタンのクリックイベントはHotで、誰が購読していようといまいと、クリックは起きています。購読者は、途中から同じ流れに加わる形です。共有された1本の管を、複数人がのぞき込むイメージです。

この違いは、「購読するたびに処理が繰り返されるのか、共有されるのか」に影響します。意図せず通信が何度も走る、といった問題は、Cold Observableの性質を理解していないことが原因のことがあります。まずは「HTTPはCold（購読ごとに走る）」という代表例を押さえておけば十分です。

## 手動での購読解除は面倒

購読の解除が大切だとわかっても、すべてを手作業で管理するのは大変です。Componentがいくつもの購読を持つと、`ngOnDestroy`で、それらをひとつずつ`unsubscribe`することになります。

```ts:手動での購読解除（煩雑な例）
export class Dashboard implements OnDestroy {
  private sub1 = this.a$.subscribe(/* ... */);
  private sub2 = this.b$.subscribe(/* ... */);

  ngOnDestroy(): void {
    this.sub1.unsubscribe();
    this.sub2.unsubscribe();
  }
}
```

購読が増えるほど、この後始末は長くなり、書き忘れのリスクも高まります。そこで、モダンAngularには、これを自動化する、より良い方法が用意されています。

## takeUntilDestroyedによる自動解除

もっとも手軽なのが、`takeUntilDestroyed()`です。これはAngularがrxjs-interopとして提供する演算子で、Componentなどが破棄されるタイミングで、購読を自動的に解除します。Angular 16で導入されました。

```ts:src/app/dashboard.ts
import { Component, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ selector: 'app-dashboard', template: `...` })
export class Dashboard {
  private readonly service = inject(DataService);

  constructor() {
    this.service.data$
      .pipe(takeUntilDestroyed())
      .subscribe((data) => {
        // dataを使った処理
      });
  }
}
```

`pipe(takeUntilDestroyed())`を挟むだけで、このComponentが破棄されるときに、購読が自動で解除されます。`ngOnDestroy`を書く必要も、Subscriptionを保持する必要もありません。書き忘れの心配がなくなるのが、大きな利点です。コンストラクター内など、注入コンテキストで呼ぶのが基本です。

## asyncパイプがもっとも簡単

そもそも、可能な場面では、`subscribe`を自分で書かないのが、もっとも安全です。第16章で学んだ`async`パイプを使えば、購読も解除もAngularが引き受けてくれます。

```html
<!-- asyncパイプ：購読も解除も自動 -->
@if (data$ | async; as data) {
  <p>{{ data.title }}</p>
}
```

テンプレートで表示するだけなら、`async`パイプがもっとも簡潔で、後始末の心配もありません。手動の`subscribe`は、テンプレートに直接表示しない処理（副作用）が必要なときの手段、と考えるとよいでしょう。そして手動で購読するときは、`takeUntilDestroyed()`で解除を自動化する。この2つを使い分ければ、購読管理の悩みの大半は解消します。

## 完了時の後始末とfinalize

購読が終わるのは、`unsubscribe`のときだけではありません。Observableが完了（complete）したり、エラーで終わったりしたときも、購読は終わります。いずれの終わり方でも、共通の後始末をしたいことがあります。そのときに使えるのが、`finalize`というOperatorです。

```ts
import { finalize } from 'rxjs';

this.loading.set(true);
this.api.getData()
  .pipe(
    finalize(() => this.loading.set(false)), // 成功・失敗・解除いずれでも実行
    takeUntilDestroyed(),
  )
  .subscribe((data) => { /* ... */ });
```

`finalize`に渡した処理は、Observableがどう終わっても（完了・エラー・購読解除のいずれでも）実行されます。この例では、通信の成否にかかわらず、ローディング表示を確実に消しています。「終わったら必ずやること」を書く場所として、`finalize`は便利です。ローディングの解除は、その典型的な使い道です。

## 状態をSignalで持てば購読管理は減る

購読管理の話をしてきましたが、そもそも手動の購読を減らすことが、根本的な対策になります。第6部で学んだSignalや、それと連携する`toSignal()`（第41章で詳しく扱います）を使えば、購読と解除をAngularに任せられます。

つまり、購読管理の悩みの多くは、「手動の`subscribe`をできるだけ書かない」ことで避けられます。表示は`async`パイプかSignalに任せ、どうしても手動購読が必要なときだけ`takeUntilDestroyed()`を添える。この方針を徹底すれば、メモリリークの心配は、ほとんどしなくて済むようになります。RxJSの購読管理は、かつてAngular開発の悩みの種でしたが、モダンな道具立てによって、大きく軽減されているのです。

## よくあるつまずき

- **購読解除の書き忘れ**: 手動`subscribe`で解除を忘れると、メモリリークの原因になります。`takeUntilDestroyed()`や`async`パイプで自動化するのが安全です。
- **`subscribe`の中で`subscribe`する**: 購読の中でさらに購読すると（ネスト）、管理が難しくなります。次章のOperator（`switchMap`など）を使えば、平らに書けます。
- **何でも手動`subscribe`にする**: 表示目的なら`async`パイプで足ります。手動購読は、副作用が必要な場面に絞ります。
- **`takeUntilDestroyed()`を注入コンテキスト外で使う**: 引数なしで使う場合は、コンストラクターなどの注入コンテキストで呼ぶ必要があります。それ以外の場所で使うには、`DestroyRef`を渡します。

## まとめ

- `subscribe`はSubscriptionを返し、`unsubscribe`で購読を終わらせます
- 購読解除を忘れると、破棄後も処理が動き続け、メモリリークを招きます
- Cold Observableは購読ごとに、Hot Observableは共有された流れとして値を流します
- モダンAngularでは、`takeUntilDestroyed()`で購読解除を自動化できます
- 表示目的なら`async`パイプがもっとも安全で、手動購読は副作用のある場面に絞ります

次章では、流れてくる値を加工・合成するOperatorを学び、非同期処理を宣言的に組み立てる方法を身につけます。
