---
title: "第20章 双方向バインディングとmodel()"
---

ここまで、親から子への入力`input()`と、子から親への出力`output()`を学びました。この2つを組み合わせると、「親から値を受け取り、子で変更して、その変更を親に返す」という双方向のやり取りが作れます。この往復を1つにまとめたのが、`model()`です。

`model()`は、Angular 17.2（2024年）で導入されました。チェックボックスやスライダー、独自の入力欄のように、値を受け取りつつ、その値を書き換えて親へ返す部品を作るのに向いています。この章では、双方向バインディングの仕組みと、`model()`の使い方を、旧来の書き方と比較しながら学びます。

:::message
**この章で学ぶこと**

- 双方向バインディング`[(value)]`の仕組み
- `model()`による双方向の入力の宣言
- `model()`が入力と出力を兼ねること
- 旧来の入力・出力ペアによる実装との違い
:::

## 双方向バインディングとは

双方向バインディングは、親と子が同じ値を共有し、どちらから変更しても、もう一方に反映される仕組みです。書き方は、角括弧と丸括弧を組み合わせた`[(value)]`です。角括弧の中に丸括弧が入った形から、バナナを箱に入れたような見た目になぞらえて「バナナ・イン・ア・ボックス」とも呼ばれます。

```html
<app-toggle [(checked)]="isDark" />
```

この`[(checked)]`は、第3部で学んだ入力`[checked]`と出力`(checkedChange)`を、同時に書いたものにあたります。つまり、双方向バインディングは、入力と出力の組み合わせを短く書くための記法なのです。親の`isDark`が子へ渡り、子が`checked`を書き換えると、その変更が親の`isDark`へ戻ります。

## model()で双方向の値を宣言する

この双方向のやり取りを子側で受け止めるのが、`model()`です。`input()`によく似ていますが、返ってくるのが読み取り専用ではなく、書き換え可能なSignalである点が異なります。

```ts:src/app/toggle.ts
import { Component, model } from '@angular/core';

@Component({
  selector: 'app-toggle',
  template: `
    <button (click)="toggle()">
      {{ checked() ? 'オン' : 'オフ' }}
    </button>
  `,
})
export class Toggle {
  checked = model(false);

  protected toggle(): void {
    this.checked.set(!this.checked());
  }
}
```

`checked = model(false)`で、双方向の値を宣言しています。値を読むときは`checked()`、書き換えるときは`checked.set()`や`checked.update()`を使います。`toggle()`で`checked`を反転させると、その変更が自動的に親へ伝わります。親は、先ほどの`[(checked)]="isDark"`で受けていれば、`isDark`が連動して変わります。

`input()`が読み取り専用だったのに対し、`model()`は子の側から値を変えられます。この「変えられる」性質が、双方向バインディングを成り立たせています。

## model()は入力と出力を兼ねる

`model()`の要点は、1つの宣言で入力と出力の両方を作り出すことです。`checked = model(false)`と書くと、Angularは内部的に次の2つを用意します。

- **`checked`という入力**: 親から値を受け取る口
- **`checkedChange`という出力**: 変更を親へ返す口

出力の名前は、モデル名に`Change`を付けたものになります。この`checked`と`checkedChange`のペアがあるからこそ、親側で`[(checked)]`という双方向の記法が使えるのです。双方向バインディング`[(checked)]`は、`[checked]`と`(checkedChange)`を同時に書いたものと等価だ、と第3部で触れたのは、このことでした。

必須にしたい場合は、`input`と同様に`model.required()`が使えます。

```ts
value = model.required<string>();
```

## 実用的な例 — 数量入力

`model()`が活きる例として、数量を増減する入力部品を作ってみます。プラスとマイナスのボタンで値を変え、その値を親と共有します。

```ts:src/app/quantity-input.ts
import { Component, model } from '@angular/core';

@Component({
  selector: 'app-quantity-input',
  template: `
    <button (click)="dec()">−</button>
    <span>{{ value() }}</span>
    <button (click)="inc()">＋</button>
  `,
})
export class QuantityInput {
  value = model(1);

  protected inc(): void {
    this.value.update((n) => n + 1);
  }

  protected dec(): void {
    this.value.update((n) => Math.max(1, n - 1));
  }
}
```

親は、この部品を`[(value)]`でつなぎ、数量を自分の状態として持てます。

```ts:src/app/cart-item.ts
@Component({
  selector: 'app-cart-item',
  imports: [QuantityInput],
  template: `
    <app-quantity-input [(value)]="quantity" />
    <p>小計: {{ subtotal() }} 円</p>
  `,
})
export class CartItem {
  protected readonly quantity = signal(1);
  protected readonly subtotal = computed(() => this.quantity() * 500);
}
```

子でボタンを押すと`quantity`が更新され、それに連動して`subtotal`が再計算されます。親は、子の内部実装（ボタンや`update`）を一切知らずに、`quantity`という値だけを共有できます。値の増減ロジックは子に閉じ、親は結果の値だけを扱う。この関心の分離が、`model()`による双方向の見通しのよさです。

ここで`update()`を使っている点にも注目してください。現在の値をもとに新しい値を計算するときは、`set()`より`update()`が簡潔です。`this.value.set(this.value() + 1)`と書く代わりに、`this.value.update((n) => n + 1)`と書けます。

## 旧来の書き方との比較

`model()`が登場する前は、双方向バインディングを作るのに、入力と出力を別々に宣言し、名前を`Change`で揃える必要がありました。同じトグルを、旧来の書き方で示します。

```ts:旧来の書き方（入力と出力のペア）
import { Component, EventEmitter, Input, Output } from '@angular/core';

@Component({ selector: 'app-toggle', template: `...` })
export class Toggle {
  @Input() checked = false;
  @Output() checkedChange = new EventEmitter<boolean>();

  toggle(): void {
    this.checked = !this.checked;
    this.checkedChange.emit(this.checked); // 変更を手作業で送出
  }
}
```

`@Input() checked`と`@Output() checkedChange`を、名前を揃えて2つ宣言します。値を変えたら、`checkedChange.emit()`を自分で呼んで、変更を親に知らせる必要がありました。この「入力と出力を2つ書く」「名前を`Change`で揃える」「emitを忘れない」という手間が、双方向バインディングの実装を煩雑にしていました。

`model()`は、これを1行にまとめます。

```ts:src/app/toggle.ts（現在の書き方）
checked = model(false);
```

入力・出力のペアの宣言も、変更時の`emit`の呼び出しも不要です。`set()`や`update()`で値を変えれば、変更は自動で親へ伝わります。記述量が大きく減り、名前の揃え忘れやemit漏れといったミスも起きなくなります。

## いつmodel()を使うか

双方向バインディングは便利ですが、どんな場面でも使うわけではありません。第17章の単方向データフローの原則は、依然として基本です。多くのやり取りは、入力`input()`と出力`output()`で素直に表現でき、そのほうがデータの流れを追いやすくなります。

`model()`が真価を発揮するのは、フォーム部品のように「値を持ち、その値を編集して返す」ことが本質的な役割である場面です。チェックボックス、スライダー、日付選択、独自の入力欄などが典型です。標準の`ngModel`（第9部で扱います）が`[(ngModel)]`で使えるのも、この双方向の仕組みによるものです。

:::message
双方向バインディングを多用すると、値がどこで変わるのかが追いにくくなることがあります。まずは`input()`・`output()`による単方向を基本とし、双方向が自然な部品にだけ`model()`を使う、という使い分けを本書では推奨します。
:::

## よくあるつまずき

`model()`と双方向バインディングでつまずきやすい点を挙げます。

- **`model()`と`input()`を取り違える**: 親から受け取るだけなら`input()`で十分です。子が値を書き換えて親へ返す必要があるときにだけ`model()`を使います。読み取るだけの値に`model()`を使うと、意図せず書き換えられる余地を残してしまいます。
- **`[( )]`の内側と外側を間違える**: 双方向バインディングは`[(value)]`と、角括弧の内側に丸括弧を書きます。順序を逆にした`([value])`は誤りです。バナナを箱に入れる形、と覚えると間違えにくくなります。
- **親側に式を渡す**: `[(value)]`の右辺には、書き換え可能な状態（Signalなど）を渡します。計算結果のような書き換えられない式を渡すと、子が値を戻せません。
- **単方向で済む場面まで双方向にする**: 表示するだけ、通知するだけのやり取りに`model()`を持ち込むと、かえって流れが見えにくくなります。双方向は、値の編集が本質の部品に限って使います。

## まとめ

- 双方向バインディング`[(value)]`は、入力`[value]`と出力`(valueChange)`を同時に書いた記法です
- `model()`は、書き換え可能なSignalとして双方向の値を宣言します
- `model()`は入力と、`Change`を付けた名前の出力を、1つの宣言で兼ねます
- 旧来は入力・出力のペアと手作業の`emit`が必要で、`model()`はそれを1行にまとめます
- **双方向が本質的なフォーム部品には`model()`を、それ以外は単方向の`input()`・`output()`を使うのが現在の標準です**

次章では、Componentの一生をたどるライフサイクルと、入力値の変化に応じた処理を学びます。
