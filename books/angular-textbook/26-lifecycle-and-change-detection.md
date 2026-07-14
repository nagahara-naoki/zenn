---
title: "第21章 ライフサイクルと入力値の変更検知"
---

第6章で、Componentには生成から破棄までの一生、すなわちライフサイクルがあると触れました。この章では、その中身を詳しく見ていきます。Componentが生まれ、入力を受け取り、画面に現れ、やがて破棄されるまでの各段階で、どんな処理を差し込めるのかを学びます。

あわせて、この部で学んだ`input()`との関わりも整理します。旧来、入力の変化を捉えるにはライフサイクルフックが欠かせませんでしたが、Signalベースの入力は、その多くを不要にします。モダンAngularでライフサイクルとどう付き合うか、その勘所をつかみましょう。

:::message
**この章で学ぶこと**

- Componentのライフサイクルフックとその順序
- `ngOnInit`と`ngOnDestroy`の使いどころ
- `ngOnChanges`と、Signal入力による代替
- DOM操作のための`afterNextRender`・`afterEveryRender`
:::

## ライフサイクルフックとは

ライフサイクルの各段階には、決められた名前のメソッドを実装することで、処理を差し込めます。これをライフサイクルフックと呼びます。Angularが適切なタイミングで、これらのメソッドを自動的に呼び出します。おもなフックを、呼ばれる順に示します。

| フック | 呼ばれるタイミング |
|---|---|
| `ngOnChanges` | 入力値が設定・変更されたとき |
| `ngOnInit` | 最初の入力設定後、初期化時に一度 |
| `ngAfterContentInit` | 投影されたコンテンツの初期化後 |
| `ngAfterViewInit` | 自身のテンプレート（子ビュー）の初期化後 |
| `ngOnDestroy` | 破棄される直前 |

このうち、日常的によく使うのは`ngOnInit`と`ngOnDestroy`です。まずはこの2つを押さえ、残りは必要になったときに理解すれば十分です。

表に挙げていないフックもあります。`ngDoCheck`は変更検知のたびに呼ばれ、`ngAfterContentChecked`・`ngAfterViewChecked`は、それぞれコンテンツ・ビューの確認のたびに呼ばれます。これらは変更検知のサイクルごとに何度も実行されるため、重い処理を書くとアプリ全体が遅くなります。特別な理由がない限り、実装する場面はまれです。フックには「一度だけ呼ばれるもの（`ngOnInit`など）」と「何度も呼ばれるもの（`ngDoCheck`や各`Checked`）」があることを意識すると、どこに何を書くべきかの判断がつきやすくなります。

## ngOnInitとngOnDestroy

`ngOnInit`は、Componentの初期化処理を書く場所です。入力値がそろった状態で一度だけ呼ばれるため、初期データの読み込みなどに使われてきました。`OnInit`インターフェースを実装して用います。

```ts:src/app/user-list.ts
import { Component, inject, OnInit } from '@angular/core';

@Component({ selector: 'app-user-list', template: `...` })
export class UserList implements OnInit {
  private readonly service = inject(UserService);

  ngOnInit(): void {
    // 初期化時にデータを読み込む
    this.service.load();
  }
}
```

`ngOnDestroy`は、その逆で、後始末を書く場所です。Componentが破棄される直前に呼ばれます。手動で開始した購読の解除や、タイマーの停止などに使います。

```ts
ngOnDestroy(): void {
  // 後始末（購読解除やタイマー停止など）
  this.subscription.unsubscribe();
}
```

なぜ後始末が必要なのでしょうか。破棄されたはずのComponentが、購読やタイマーを通じて動き続けると、メモリリークや意図しない処理の原因になります。`ngOnDestroy`で確実に片づけることが、健全なアプリケーションの条件です。RxJSの購読解除については、第8部で詳しく扱います。

## ngOnChangesとSignal入力

`ngOnChanges`は、入力値が変わるたびに呼ばれるフックです。旧来の`@Input`では、入力の変化に応じた処理を、ここに書いていました。

```ts:旧来の書き方（ngOnChangesで入力変化に反応）
import { Component, Input, OnChanges } from '@angular/core';

@Component({ selector: 'app-price-tag', template: `...` })
export class PriceTag implements OnChanges {
  @Input() price = 0;
  withTax = 0;

  ngOnChanges(): void {
    // priceが変わるたびに、手作業で再計算する
    this.withTax = Math.floor(this.price * 1.1);
  }
}
```

`price`が変わるたびに`withTax`を計算し直す、という処理です。入力の数が増えると、この`ngOnChanges`は肥大化しがちでした。

第18章で見たように、`input()`を使うと、この多くが不要になります。入力がSignalなので、派生値は`computed()`で宣言でき、変化への追従は自動で行われます。

```ts:src/app/price-tag.ts（現在の書き方）
import { Component, computed, input } from '@angular/core';

@Component({ selector: 'app-price-tag', template: `...` })
export class PriceTag {
  price = input(0);
  protected readonly withTax = computed(() => Math.floor(this.price() * 1.1));
}
```

`ngOnChanges`も、その中の再計算も要りません。`computed()`が、`price`の変化を検知して自動で再計算します。Signal入力の変化に応じて副作用を起こしたい場合は、`effect()`（第6部で扱います）を使えます。このように、モダンAngularでは、入力の変化にまつわるライフサイクルフックの出番が、大きく減っています。

## ビューの初期化フック

`ngAfterViewInit`は、Componentのテンプレート（子ビュー）が初期化された後に呼ばれます。`ngAfterContentInit`は、投影されたコンテンツ（第8章）が初期化された後です。これらは、子要素への参照を扱うために使われてきました。

もっとも、第8章で学んだ`viewChild()`・`contentChild()`は、Signalベースのクエリです。参照はSignalとして得られ、値がそろえば自動で反映されます。そのため、`ngAfterViewInit`を実装して参照を取りにいく、という旧来のパターンも、必要な場面が減っています。ここでも、Signalがライフサイクルフックを肩代わりしているのです。

## DOMを扱うafterNextRenderとafterEveryRender

ときには、Angularの管理の外にあるDOMを直接操作したいことがあります。たとえば、描画された要素の実寸を測る、外部のJavaScriptライブラリを要素に適用する、といった処理です。こうした「描画が終わった後にDOMを触る」用途のために、`afterNextRender`と`afterEveryRender`という関数が用意されています。

```ts:src/app/chart.ts
import { Component, afterNextRender, ElementRef, inject } from '@angular/core';

@Component({ selector: 'app-chart', template: `<div #box></div>` })
export class Chart {
  private readonly host = inject(ElementRef);

  constructor() {
    afterNextRender(() => {
      // 描画完了後に一度だけDOMを操作する
      const width = this.host.nativeElement.offsetWidth;
      console.log(`幅: ${width}`);
    });
  }
}
```

`afterNextRender`は次の描画完了後に一度だけ、`afterEveryRender`は描画のたびに呼ばれます。これらはコンストラクター内で登録します。なお、`afterEveryRender`は、Angular 20（2025年）で`afterRender`から改称されたものです。少し前のコードや記事では`afterRender`という名前で登場することがあります。

これらの関数は、サーバーサイドレンダリング（第62章）の環境ではブラウザでのみ実行されるよう配慮されており、DOM操作を安全に書けます。とはいえ、DOMの直接操作は最小限にとどめ、まずはテンプレートとバインディングで表現できないかを検討するのが基本です。

## よくあるつまずき

- **`ngOnInit`とコンストラクターの混同**: コンストラクターは、依存の注入（`inject()`）など、生成時の準備に使います。入力値を使う初期化は、値がそろう`ngOnInit`で行うのが安全です。ただし`input()`とSignalを使えば、多くはそもそもフックなしで書けます。
- **`ngOnDestroy`での後始末忘れ**: 手動の購読やタイマーは、`ngOnDestroy`で片づけないと残り続けます。`async`パイプ（第16章）や`takeUntilDestroyed`（第8部）を使うと、この後始末を自動化できます。
- **不要なライフサイクルフックの実装**: Signalで書ける処理を、習慣で`ngOnChanges`や`ngAfterViewInit`に書いてしまうことがあります。モダンAngularでは、まず`computed()`・`effect()`・Signalクエリで書けないかを考えます。

## まとめ

- Componentには初期化から破棄までのライフサイクルがあり、フックで処理を差し込めます
- よく使うのは初期化の`ngOnInit`と、後始末の`ngOnDestroy`です
- 入力変化の`ngOnChanges`は、`input()`＋`computed()`で多くが不要になります
- 子要素参照の`ngAfterViewInit`も、Signalクエリ（`viewChild()`）で代替できる場面が増えています
- 描画後のDOM操作には`afterNextRender`・`afterEveryRender`を使います

以上で第4部は終わりです。次の第5部では、Componentから処理を切り出して受け持つServiceと、それを支えるDependency Injection（依存性の注入）を学びます。
