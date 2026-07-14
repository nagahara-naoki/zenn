---
title: "第8章 コンテンツ投影とクエリ — ng-contentとviewChild()・contentChild()"
---

前章までで、Componentを作り、組み合わせる方法を学びました。この章では、Componentをより柔軟に再利用するための2つの仕組みを扱います。ひとつは、Componentの外側から中身を差し込む「コンテンツ投影」。もうひとつは、Component内の子要素をクラスから参照する「クエリ」です。

この2つは、一見すると別々の話題に見えますが、深く関わっています。コンテンツ投影で差し込まれた要素を、クエリで参照する、という場面がよくあるためです。まずコンテンツ投影から見ていきましょう。

:::message
**この章で学ぶこと**

- `ng-content`によるコンテンツ投影の仕組み
- 複数スロットとフォールバックコンテンツ
- `viewChild()`・`contentChild()`による子要素の参照
- 旧来の`@ViewChild`・`@ContentChild`との違い
:::

## コンテンツ投影とは

カードやダイアログのような「枠」を作ることを考えます。枠の見た目は共通で、中身だけが場面ごとに異なる、という部品です。このとき、中身をComponentの外側から渡せると便利です。この仕組みがコンテンツ投影で、`ng-content`を使います。

次の`Card`は、外から渡された内容を、枠の中に差し込むComponentです。

```ts:src/app/card.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <ng-content />
    </div>
  `,
  styleUrl: './card.css',
})
export class Card {}
```

この`Card`を使う側は、開始タグと終了タグの間に中身を書きます。その中身が、`<ng-content />`の位置に差し込まれます。

```html
<app-card>
  <p>ここに書いた内容がカードの中に表示されます</p>
</app-card>
```

`Card`自身は中身を知りません。中身を決めるのは使う側です。これにより、枠は共通のまま、中身だけを場面ごとに変えられます。

## 複数のスロットに振り分ける

差し込み先を複数に分けたいこともあります。たとえば「見出し」と「本文」を別々の位置に置きたい場合です。このときは、`ng-content`に`select`属性を付けて、どの内容をどこに差し込むかを指定します。

```ts:src/app/card.ts
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <header><ng-content select="[card-title]" /></header>
      <div class="body"><ng-content /></div>
    </div>
  `,
})
export class Card {}
```

`select="[card-title]"`は、「`card-title`属性を持つ要素をここに差し込む」という指定です。`select`はComponentのセレクターと同じCSSセレクターを使えるため、属性・要素名・クラスなどで振り分けられます。使う側は、次のように書きます。

```html
<app-card>
  <h2 card-title>お知らせ</h2>
  <p>本文がここに入ります</p>
</app-card>
```

`card-title`属性が付いた`<h2>`は見出しの位置へ、それ以外の`<p>`は`select`のない`ng-content`（本文）へ振り分けられます。

:::message
`select`のない`<ng-content />`が1つもない場合、どのスロットにも一致しなかった内容は、画面に表示されません。振り分けと「その他」の受け皿を意識して設計してください。
:::

## フォールバックコンテンツ

外から何も渡されなかったときに備えて、既定の内容を用意できます。`ng-content`の中に書いた内容が、そのフォールバック（代替）になります。

```ts
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <ng-content select="[card-title]">タイトルなし</ng-content>
    </div>
  `,
})
export class Card {}
```

使う側が`card-title`の要素を渡さなかった場合、「タイトルなし」が代わりに表示されます。渡された場合は、そちらが優先されます。この`ng-content`内にフォールバックを書ける仕組みは、Angular 18で導入されました。

## ngProjectAsと投影の注意点

要素を、実際のタグとは別のセレクターとして振り分けたいときは、`ngProjectAs`属性を使います。

```html
<app-card>
  <h2 ngProjectAs="[card-title]">見出し</h2>
</app-card>
```

これで`<h2>`は、`card-title`属性を持つものとして扱われ、見出しのスロットへ差し込まれます。

コンテンツ投影で注意したいのは、`ng-content`はビルド時に処理され、投影はComponentの生成時に一度だけ行われるという点です。`ng-content`自体を`@if`で囲んで出し分けるような使い方は想定されていません。表示を条件で切り替えたい場合は、投影された内容ではなく、枠側の構造で工夫します。

## 子要素を参照する — クエリ

ここからは、Componentのクラスから、テンプレート内の子要素を参照する「クエリ」を扱います。モダンAngularでは、Signalとして結果を受け取る`viewChild()`・`contentChild()`などの関数を使います。

参照先には、2つの種類があります。この違いがクエリを理解する鍵です。

- **ビュー（view）**: そのComponent自身のテンプレートに書かれた要素
- **コンテンツ（content）**: 外から`ng-content`に差し込まれた要素

自身のテンプレート内を参照するのが`viewChild()`、投影された内容を参照するのが`contentChild()`です。

### viewChild() — 自身のテンプレートを参照する

`viewChild()`は、そのComponentのテンプレートにある子Componentや要素を参照します。結果はSignalなので、`()`を付けて値を取り出します。

```ts:src/app/card-list.ts
import { Component, viewChild, computed } from '@angular/core';
import { CardHeader } from './card-header';

@Component({
  selector: 'app-card-list',
  imports: [CardHeader],
  template: `<app-card-header />`,
})
export class CardList {
  header = viewChild(CardHeader);
  headerText = computed(() => this.header()?.text);
}
```

結果は「まだ見つかっていない」状態もありうるため、値は`undefined`の可能性を含みます。必ず存在すると保証したい場合は、`.required()`を使います。見つからなければAngularがエラーを報告し、戻り値から`undefined`が除かれます。

```ts
header = viewChild.required(CardHeader);
```

### contentChild() — 投影された内容を参照する

`contentChild()`は、`ng-content`を通して外から差し込まれた要素を参照します。書き方は`viewChild()`とまったく同じで、参照する対象が「投影されたコンテンツ」である点だけが異なります。

```ts:src/app/card.ts
import { Component, contentChild } from '@angular/core';
import { CardTitle } from './card-title';

@Component({
  selector: 'app-card',
  template: `<div class="card"><ng-content /></div>`,
})
export class Card {
  title = contentChild(CardTitle);
}
```

複数の要素をまとめて参照したいときは、`viewChildren()`・`contentChildren()`を使います。これらは配列のSignalを返します。参照範囲を孫要素まで広げたい場合は、`contentChildren(CardTitle, { descendants: true })`のようにオプションを指定します。

### 旧来の@ViewChild・@ContentChildとの違い

Signalベースのクエリが登場する前は、`@ViewChild`・`@ContentChild`というデコレーターを使っていました。

```ts:旧来の書き方（デコレーター）
export class CardList {
  @ViewChild(CardHeader) header!: CardHeader;

  ngAfterViewInit(): void {
    console.log(this.header.text);
  }
}
```

旧来の書き方では、参照結果を安全に扱えるのが`ngAfterViewInit`のようなライフサイクルの後になる、という制約がありました。結果を取り出すタイミングを、開発者が意識する必要があったのです。

一方、Signalベースの`viewChild()`は結果をSignalとして返すため、`computed()`や`effect()`と自然に組み合わせられ、ライフサイクルを意識せずに扱えます。`viewChild()`はAngular 17.2でプレビュー導入され、Angular 19で安定版になりました。

### よくある混同 — viewとcontentの取り違え

`viewChild()`と`contentChild()`は書き方がそっくりなので、取り違えやすい点に注意してください。判断の基準は「その要素が、自分のテンプレートに直接書かれているか、それとも`ng-content`を通して外から差し込まれたか」です。自分のテンプレートに書いた子Componentは`viewChild()`で、`<app-card>ここ</app-card>`のように外から渡された内容は`contentChild()`で参照します。`contentChild()`で自分のテンプレート内を探しても、`viewChild()`で投影された内容を探しても、対象は見つかりません。参照したい要素が「どちら側にあるか」を意識すると、迷わなくなります。

## まとめ

- コンテンツ投影は`ng-content`で実現し、Componentの外側から中身を差し込めます
- `select`属性で複数スロットに振り分け、`ng-content`内にフォールバックを書けます
- クエリには、自身のテンプレートを見る`viewChild()`と、投影された内容を見る`contentChild()`があります
- クエリの結果はSignalで返り、`.required()`で存在を保証できます
- **旧来の`@ViewChild`・`@ContentChild`に代えて、現在はSignalベースの`viewChild()`・`contentChild()`を使うのが推奨です**

次章では、Componentのスタイルがどのように分離されるのか、その仕組みであるView Encapsulationを学びます。
