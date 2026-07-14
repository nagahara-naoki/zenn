---
title: "第15章 構造Directiveとng-templateの仕組み"
---

前章では、既存の要素に振る舞いを足す属性Directiveを作りました。この章では、もうひとつの種類である構造Directiveを扱います。構造Directiveは、DOMの構造そのもの、つまり要素の追加や削除を担うDirectiveです。

第12章で学んだ`@if`・`@for`は、いまや制御フローの主役ですが、その前身である`*ngIf`・`*ngFor`は構造Directiveでした。この章では、構造Directiveがどのようにテンプレートを操作しているのか、その土台にある`ng-template`とあわせて解き明かします。仕組みを理解すると、テンプレートの一部を変数のように扱う高度な表現や、既存コードの読解ができるようになります。

:::message
**この章で学ぶこと**

- 構造Directiveが何をするか
- `*`記法と`ng-template`の関係
- `TemplateRef`と`ViewContainerRef`の役割
- `ng-template`と`ngTemplateOutlet`の使い方
:::

## 構造Directiveとは

構造Directiveは、DOMに要素を追加したり、取り除いたりするDirectiveです。属性Directiveが「すでにある要素の見た目や振る舞いを変える」のに対し、構造Directiveは「要素そのものを出したり消したりする」点が異なります。

旧来の`*ngIf`を例に見てみます。

```html
<p *ngIf="isVisible()">条件が真のときだけ表示される段落</p>
```

`isVisible()`が`true`のときは`<p>`がDOMに現れ、`false`のときはDOMから消えます。表示・非表示をCSSで切り替えるのではなく、要素そのものをDOMに存在させたり、させなかったりするのが構造Directiveの特徴です。この先頭の`*`が、構造Directiveであることの目印です。

## *記法はng-templateの短縮形

この`*`という記法には、からくりがあります。`*`は、`ng-template`という要素を使った書き方の、短縮形（糖衣構文）なのです。先ほどの`*ngIf`は、内部的には次のように展開されます。

```html
<!-- *ngIf は、この形の短縮形 -->
<ng-template [ngIf]="isVisible()">
  <p>条件が真のときだけ表示される段落</p>
</ng-template>
```

`ng-template`は、「描画されずに保持されるテンプレートの断片」を表す要素です。中に書いた内容は、すぐには画面に出ません。構造Directiveが「いま表示すべき」と判断したときに、はじめてDOMへ差し込まれます。`*ngIf`は、この`ng-template`の中身を、条件に応じて出し入れしていたわけです。

`*`記法は、この`ng-template`の記述を省いて簡潔に書けるようにしたものです。1つの要素に付けられる構造Directiveは1つだけ、という制約があるのも、この展開の仕組みに由来します。

## ng-templateとは

`ng-template`は、それ単体では何も描画しません。あくまで「あとで使うためのテンプレートの入れ物」です。次のコードでは、`ng-template`の中身は画面に現れません。

```html
<ng-template>
  <p>この段落は、そのままでは表示されません</p>
</ng-template>
```

保持されたテンプレートを実際に画面へ出すには、それを差し込む指示が必要です。その指示を出すのが構造Directiveであり、指示の対象となるのが、次に説明する`TemplateRef`と`ViewContainerRef`です。

## TemplateRefとViewContainerRef

構造Directiveを自作するには、2つの部品を理解する必要があります。

- **`TemplateRef`**: `ng-template`が保持しているテンプレートの断片への参照です。「何を差し込むか」を表します。
- **`ViewContainerRef`**: テンプレートを差し込む先の、DOM上の入れ物です。「どこに差し込むか」を表します。

構造Directiveは、この2つを組み合わせて動きます。`ViewContainerRef`に対して「`TemplateRef`の中身を描画せよ」と命じれば要素が現れ、「消せ」と命じれば要素が消えます。`*ngIf`が行っていたのは、まさにこの操作でした。

## 自作の構造Directiveを作る

仕組みを理解するために、`*ngIf`の逆、つまり「条件が偽のときだけ表示する」`appUnless`を作ってみます。

```ts:src/app/unless.ts
import {
  Directive,
  effect,
  inject,
  input,
  TemplateRef,
  ViewContainerRef,
} from '@angular/core';

@Directive({
  selector: '[appUnless]',
})
export class Unless {
  private readonly templateRef = inject(TemplateRef);
  private readonly viewContainer = inject(ViewContainerRef);

  readonly appUnless = input.required<boolean>();

  constructor() {
    effect(() => {
      this.viewContainer.clear();
      if (!this.appUnless()) {
        this.viewContainer.createEmbeddedView(this.templateRef);
      }
    });
  }
}
```

`inject(TemplateRef)`で「付けられた要素のテンプレート」を、`inject(ViewContainerRef)`で「その差し込み先」を取得しています。`appUnless`の値を`effect()`で監視し、偽になったら`createEmbeddedView`でテンプレートを描画し、真なら`clear`で消しています。使う側は`*ngIf`と同じ感覚で書けます。

```html
<p *appUnless="isLoggedIn()">ログインしていません</p>
```

`isLoggedIn()`が偽のときだけ、段落が表示されます。`*appUnless`が、先ほど説明した`ng-template`版へ展開され、`TemplateRef`と`ViewContainerRef`を通してDOMが操作されているのです。

## ngTemplateOutletでテンプレートを差し込む

`ng-template`は、構造Directive以外にも活用できます。保持したテンプレートを、任意の場所へ差し込む`ngTemplateOutlet`です。これを使うと、テンプレートの断片を変数のように扱えます。

```html
<ng-template #greeting>
  <p>こんにちは</p>
</ng-template>

<div>
  <ng-container [ngTemplateOutlet]="greeting" />
</div>
```

`#greeting`という参照変数で`ng-template`に名前を付け、`[ngTemplateOutlet]="greeting"`でその中身を差し込んでいます。同じテンプレートを複数の場所で使い回したり、条件に応じて差し込むテンプレートを切り替えたりできます。ここで使った`ng-container`は、DOMに余計な要素を残さずにグループ化するための、目印のないタグです。

差し込むテンプレートに、値を渡すこともできます。`ng-template`側で受け取る変数を宣言し、`ngTemplateOutletContext`で値を渡します。

```html
<ng-template #row let-name="name">
  <p>名前: {{ name }}</p>
</ng-template>

<ng-container
  [ngTemplateOutlet]="row"
  [ngTemplateOutletContext]="{ name: '山田' }"
/>
```

`let-name="name"`は、渡されたコンテキストの`name`を、テンプレート内で`name`という変数として使う宣言です。この仕組みを応用すると、「一覧の各行の見た目を、使う側から差し替えられるComponent」のような、柔軟な部品を作れます。

## 制御フロー時代の構造Directive

第12章で見たとおり、条件分岐や繰り返しという構造Directiveの代表的な用途は、いまや`@if`・`@for`という組み込み制御フローが担っています。そのため、`*ngIf`のような構造Directiveを日常的に書く場面や、自作する機会は、以前より大きく減りました。

とはいえ、`ng-template`と`ngTemplateOutlet`は現在も有効です。再利用可能なテンプレートの断片を扱う場面や、Componentに「差し込むテンプレート」を外から渡す設計では、いまも使われます。また、既存コードには`*ngIf`をはじめとする構造Directiveが数多く残っています。仕組みを理解しておくことは、そうしたコードを読み解くうえでも役立ちます。

:::message
新しく条件分岐や繰り返しを書くときは、構造Directiveではなく`@if`・`@for`を使います。構造Directiveの仕組みは、`ng-template`を用いた高度なテンプレート操作と、既存コードの読解のために押さえておく知識と位置づけてください。
:::

## よくあるつまずき

構造Directiveと`ng-template`まわりでつまずきやすい点を挙げます。

- **1つの要素に構造Directiveを2つ付ける**: `*`記法は`ng-template`への展開を伴うため、1要素に付けられる構造Directiveは1つだけです。旧来、`*ngIf`と`*ngFor`を同じ要素に付けられなかったのはこのためです。`ng-container`ではさんで階層を分けるか、組み込み制御フローを使います。
- **`ng-template`の中身が表示されない**: `ng-template`は、そのままでは描画されません。表示するには構造Directiveか`ngTemplateOutlet`による差し込みが必要です。「書いたのに出ない」と感じたら、この点を確認します。
- **`ng-container`と`ng-template`の混同**: `ng-container`は、余計なDOMを作らずに要素をまとめるための入れ物で、中身はそのまま描画されます。`ng-template`は、差し込まれるまで描画されない保持用の入れ物です。役割が異なる点に注意します。

## まとめ

- 構造Directiveは、DOMに要素を追加・削除するDirectiveで、先頭の`*`が目印です
- `*`記法は`ng-template`を使った書き方の短縮形で、テンプレートの断片を出し入れします
- `TemplateRef`は差し込む内容、`ViewContainerRef`は差し込む先を表します
- `ng-template`と`ngTemplateOutlet`で、テンプレートの断片を再利用できます
- 条件分岐・繰り返しは`@if`・`@for`が標準となり、構造Directiveの自作機会は減っています

次章では、テンプレート内で値を整形するもうひとつの道具、Pipeを学びます。
