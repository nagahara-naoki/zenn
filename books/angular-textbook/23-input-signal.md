---
title: "第18章 @Inputからinput()へ"
---

前章で、親から子へデータを渡す「入力」の役割を確認しました。この章では、その入力を宣言する具体的なAPIを学びます。現在の標準は、Signalベースの`input()`関数です。一方、少し前のコードでは`@Input`デコレーターが使われていました。両者を比較しながら、`input()`の書き方と利点を掘り下げます。

`input()`は、Angular 17.1（2024年）で安定版になった、比較的新しいAPIです。Signalとして入力を扱えるため、値の変化に応じた処理が書きやすく、`@Input`が抱えていたいくつかの課題を解消しています。既存コードには`@Input`が数多く残っているため、両方を理解しておくことが大切です。

:::message
**この章で学ぶこと**

- `input()`による入力の宣言
- 必須入力・既定値・別名・変換
- 入力から派生した値を`computed()`で作る方法
- 旧来の`@Input`との違い
:::

## input()で入力を宣言する

親からデータを受け取るには、`input()`で入力を宣言します。返ってくるのは読み取り専用のSignalです。

```ts:src/app/user-badge.ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-user-badge',
  template: `<span>{{ name() }}</span>`,
})
export class UserBadge {
  name = input('ゲスト');
}
```

`name = input('ゲスト')`で、`name`という入力を定義しています。引数の`'ゲスト'`は既定値で、親が値を渡さなかったときに使われます。テンプレートでは`name()`と、Signalとして`()`を付けて値を読み取ります。親からは、プロパティバインディングで渡します。

```html
<app-user-badge [name]="userName()" />
```

入力がSignalであることが、`input()`の核心です。値が変わると、それを使っている`computed()`やテンプレートが自動で反応します。第3部で学んだSignalの仕組みが、そのまま入力にも通じるのです。

## 必須の入力

値が必ず渡されるべき入力は、`input.required()`で宣言します。既定値を持たず、親が渡さないとコンパイル時にエラーになります。

```ts:src/app/user-badge.ts
export class UserBadge {
  userId = input.required<number>();
}
```

`required`には既定値がないため、型を型引数`<number>`で明示します。「このComponentは、この値がないと成り立たない」という前提を、型の力で保証できます。渡し忘れを実行前に検出できるため、安全性が高まります。

## 別名と変換

入力には、いくつかの調整用のオプションがあります。よく使うのが、別名（alias）と変換（transform）です。

別名は、テンプレートで使う名前を、クラスのプロパティ名と別にしたいときに使います。

```ts
value = input(0, { alias: 'sliderValue' });
```

```html
<app-slider [sliderValue]="50" />
```

変換は、受け取った値をクラスに取り込む前に加工したいときに使います。たとえば、文字列の前後の空白を除く、といった処理です。

```ts:src/app/text-field.ts
function trimString(value: string | undefined): string {
  return value?.trim() ?? '';
}

export class TextField {
  label = input('', { transform: trimString });
}
```

`transform`を指定すると、親から渡された値が`trimString`を通ってから`label`に格納されます。真偽値への変換など、定型的な変換はAngularが用意した関数（`booleanAttribute`・`numberAttribute`）も使えます。

`booleanAttribute`は、標準のHTML属性のように「属性が付いていれば`true`」という挙動を実現します。たとえば`disabled`のような入力を作るときに便利です。

```ts:src/app/action-button.ts
import { booleanAttribute, Component, input } from '@angular/core';

@Component({ selector: 'app-action-button', template: `...` })
export class ActionButton {
  disabled = input(false, { transform: booleanAttribute });
}
```

こうすると、`<app-action-button disabled />`のように属性を書くだけで`true`とみなされ、標準のボタンと同じ感覚で扱えます。`numberAttribute`も同様に、文字列で渡された値を数値へ変換します。定型の変換は自前で書かず、これらの用意された関数を使うのが簡潔です。

## 入力から派生した値を作る

`input()`がSignalであることの利点が、もっともよく表れるのが、入力から別の値を導く場面です。第6部で詳しく扱う`computed()`を使うと、入力の変化に追従する派生値を、宣言的に書けます。

```ts:src/app/price-tag.ts
import { Component, computed, input } from '@angular/core';

@Component({
  selector: 'app-price-tag',
  template: `<p>税込 {{ withTax() }} 円</p>`,
})
export class PriceTag {
  price = input.required<number>();
  protected readonly withTax = computed(() => Math.floor(this.price() * 1.1));
}
```

`withTax`は、`price`が変わるたびに自動で再計算されます。`price`という入力Signalを`computed()`の中で読んでいるため、両者が連動するのです。これを`@Input`でやろうとすると、次に見るように、値の変化を検知するライフサイクルフックが必要でした。`input()`は、その手間をなくします。

## 旧来の@Inputとの比較

`input()`が登場する前は、`@Input`デコレーターで入力を宣言していました。同じ`name`入力を、旧来の書き方で示します。

```ts:旧来の書き方（@Inputデコレーター）
import { Component, Input } from '@angular/core';

@Component({ selector: 'app-user-badge', template: `<span>{{ name }}</span>` })
export class UserBadge {
  @Input() name = 'ゲスト';
}
```

`@Input()`を付けたプロパティが、そのまま入力になります。テンプレートでは`{{ name }}`と、ふつうのプロパティとして読みます。一見シンプルですが、いくつかの課題がありました。

- **変化の検知が面倒**: 入力が変わったときに処理をしたい場合、`ngOnChanges`というライフサイクルフックを実装する必要がありました。派生値の計算も、この中で手作業で行っていました。
- **必須の入力を保証しにくい**: `@Input`には、値が必ず渡されることを型で強制する手段が、標準ではありませんでした。
- **Signalと連携しにくい**: 値がただのプロパティなので、Signalベースの`computed()`や`effect()`と自然にはつながりませんでした。

`input()`は、これらをまとめて解決します。入力がSignalなので、変化の検知は`computed()`や`effect()`に任せられ、`ngOnChanges`が不要になります。必須入力は`input.required()`で型として保証できます。両者の違いを表に整理します。

| 観点 | 旧来の`@Input` | 現在の`input()` |
|---|---|---|
| 宣言 | `@Input() name = ''` | `name = input('')` |
| 値の読み取り | `this.name` | `this.name()`（Signal） |
| 必須の強制 | 標準では難しい | `input.required()` |
| 変化への追従 | `ngOnChanges`を実装 | `computed()`・`effect()`が自動追従 |
| 派生値 | フック内で手作業 | `computed()`で宣言的に |

表からわかるように、`input()`は「値がSignalである」という一点から、必須の保証や自動追従といった利点が生まれています。宣言そのものは`@Input`とほぼ同じ手軽さでありながら、後段の扱いやすさが大きく向上しているのです。

:::message
既存プロジェクトの`@Input`を`input()`へ書き換える、公式の移行スキマティクスも用意されています。`ng generate @angular/core:signal-input-migration`で実行でき、段階的な移行が可能です。
:::

## よくあるつまずき

`input()`を使い始めるときに、つまずきやすい点を挙げます。

- **`()`の付け忘れ**: `input()`はSignalを返すため、値を読むときは`name()`と括弧が必要です。`{{ name }}`と書くと、値ではなく関数が表示されてしまいます。
- **入力を書き換えようとする**: `input()`が返すのは読み取り専用のSignalです。子の側で値を変えることはできません。これは、前章の単方向データフローの原則に沿ったものです。書き換えたい場合は、第20章の`model()`を使います。
- **`required`に既定値を渡す**: `input.required()`は既定値を持てません。型引数で型だけを指定します。
- **コンストラクターで入力値を読む**: 入力の値は、コンストラクターの時点ではまだ設定されていないことがあります。入力に依存する処理は、`computed()`や`effect()`の中で読むと、値がそろってから安全に扱えます。

## まとめ

- `input()`は、親から子へのデータを受け取る、Signalベースの入力宣言です
- `input.required()`で必須の入力を、`alias`で別名を、`transform`で変換を指定できます
- 入力から派生した値は`computed()`で宣言的に作れ、`ngOnChanges`が不要になります
- 旧来の`@Input`は、変化の検知に`ngOnChanges`を要し、Signalとも連携しにくいものでした
- **新規開発では`input()`を使うのが現在の標準です。`@Input`は既存コードの理解のために押さえます**

次章では、子から親へ出来事を伝える出力を、旧来の`@Output`と現在の`output()`を比較しながら学びます。
