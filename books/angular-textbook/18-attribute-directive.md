---
title: "第14章 属性Directiveを作成する"
---

前章で、Directiveには属性Directiveと構造Directiveがあり、属性Directiveは既存の要素に振る舞いを足すものだと学びました。この章では、その属性Directiveを実際に自分で作ります。

題材として、「マウスが乗ったら背景色を変える」ハイライトDirectiveを段階的に組み立てます。この過程で、ホスト要素の取得、イベントやプロパティのバインディング、そして外から値を受け取る`input()`の使い方を学びます。属性Directiveの作り方が分かると、Component間で共通する振る舞いを部品として切り出せるようになります。

:::message
**この章で学ぶこと**

- `@Directive`による属性Directiveの作り方
- ホスト要素の取得と操作
- `host`によるイベント・プロパティのバインディング
- `input()`でDirectiveに値を渡す方法
:::

## 属性Directiveの基本形

属性Directiveは、`@Directive`デコレーターを付けたクラスとして作ります。Angular CLIで生成できます。

```bash
ng generate directive highlight
```

生成される基本形は、次のようになります。

```ts:src/app/highlight.ts
import { Directive } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
})
export class Highlight {}
```

注目すべきは`selector`です。角括弧で囲んだ`[appHighlight]`は、「`appHighlight`という属性を持つ要素」に適用される、という意味です。第6章で学んだ属性型セレクターがこれにあたります。テンプレートでは、次のように既存の要素へ属性として付けます。

```html
<p appHighlight>この段落をハイライトします</p>
```

セレクターの`app`という接頭辞は、Componentのときと同じく、自作のDirectiveであることを示す目印です。標準の属性や外部ライブラリとの衝突を避けます。

## ホスト要素を操作する

Directiveが付けられた要素を、ホスト要素と呼びます。ホスト要素を操作するには、`inject()`で`ElementRef`を取り出します。`ElementRef`は、そのDOM要素への参照です。

```ts:src/app/highlight.ts
import { Directive, ElementRef, inject } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
})
export class Highlight {
  private readonly el = inject(ElementRef);

  constructor() {
    this.el.nativeElement.style.backgroundColor = 'yellow';
  }
}
```

`this.el.nativeElement`が、ホストのDOM要素そのものです。ここではその背景色を黄色にしています。これで、`appHighlight`を付けた要素の背景が黄色になります。

ただし、`nativeElement`を直接操作する書き方は、そのままでも動きますが、次に見る`host`を使うほうが宣言的で読みやすくなります。

## hostでイベントとプロパティをバインドする

ハイライトDirectiveを、「マウスが乗ったときだけ色を変える」ように改良します。ここで使うのが、`@Directive`の`host`プロパティです。`host`には、ホスト要素へのイベントリスナーやプロパティバインディングを、まとめて宣言できます。

```ts:src/app/highlight.ts
import { Directive } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  host: {
    '(mouseenter)': 'onEnter()',
    '(mouseleave)': 'onLeave()',
    '[style.backgroundColor]': 'background',
  },
})
export class Highlight {
  protected background = '';

  protected onEnter(): void {
    this.background = 'yellow';
  }

  protected onLeave(): void {
    this.background = '';
  }
}
```

`host`の中身を見てみましょう。

- `'(mouseenter)': 'onEnter()'` は、ホスト要素の`mouseenter`イベントで`onEnter`を呼ぶ、というイベントバインディングです。
- `'[style.backgroundColor]': 'background'` は、ホスト要素の背景色を`background`プロパティに結びつける、というプロパティバインディングです。

テンプレートで書いた`(event)`や`[prop]`と、同じ記法がそのまま使えます。マウスが乗ると`background`が`yellow`になり、離れると空になって、背景色が切り替わります。`ElementRef`を直接触らずに、宣言的に振る舞いを書けるのが利点です。

## 旧来のHostListener・HostBindingとの比較

`host`プロパティは、比較的新しい書き方です。少し前のコードでは、同じことを`@HostListener`と`@HostBinding`というデコレーターで書いていました。

```ts:旧来の書き方（HostListener・HostBinding）
import { Directive, HostBinding, HostListener } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
})
export class Highlight {
  @HostBinding('style.backgroundColor') background = '';

  @HostListener('mouseenter') onEnter(): void {
    this.background = 'yellow';
  }

  @HostListener('mouseleave') onLeave(): void {
    this.background = '';
  }
}
```

`@HostListener`がイベント、`@HostBinding`がプロパティのバインディングに対応します。動作は`host`プロパティ版とまったく同じです。現在のAngular公式ドキュメントでは、バインディングを1か所にまとめられる`host`プロパティが標準として示されています。既存コードで`@HostListener`・`@HostBinding`を見かけたら、`host`と同じ役割だと理解してください。

## input()でDirectiveに値を渡す

ハイライトの色を、使う側から指定できるようにします。Directiveは、Componentと同じく`input()`で外から値を受け取れます。

```ts:src/app/highlight.ts
import { Directive, input } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  host: {
    '(mouseenter)': 'onEnter()',
    '(mouseleave)': 'onLeave()',
    '[style.backgroundColor]': 'background',
  },
})
export class Highlight {
  readonly color = input('yellow');
  protected background = '';

  protected onEnter(): void {
    this.background = this.color();
  }

  protected onLeave(): void {
    this.background = '';
  }
}
```

`color = input('yellow')`で、`color`という入力を定義しています。引数の`'yellow'`は既定値です。使う側は、次のように色を渡せます。

```html
<p appHighlight color="lightblue">水色でハイライト</p>
<p appHighlight>既定の黄色でハイライト</p>
```

さらに、Directive名と同じ名前の入力を作ると、属性への代入で直接値を渡せます。セレクターと同名の入力に`alias`を使う書き方です。

```ts
readonly appHighlight = input('yellow');
```

```html
<p [appHighlight]="'lightblue'">水色でハイライト</p>
```

これで、`appHighlight`という属性の付与と、色の指定を1つにまとめられます。組み込みの`ngClass`が`[ngClass]="..."`と書けるのも、この仕組みによるものです。

## セレクターと命名規則

Directiveのセレクターには、いくつかの慣習があります。

- **接頭辞を付ける**: `app`のような接頭辞で、自作Directiveであることを示します。標準属性や他ライブラリとの衝突を防ぎます。
- **キャメルケースで書く**: 属性型セレクターは`appHighlight`のように、先頭を小文字にしたキャメルケースで書きます。HTMLの属性名として自然になります。
- **意味を表す名前にする**: `appHighlight`・`appAutoFocus`のように、振る舞いが分かる名前を付けます。

これらは、Componentの要素型セレクターがケバブケースだったのと対照的です。属性として付ける都合上、Directiveはキャメルケースが基本になります。

## よくあるつまずき

属性Directiveでつまずきやすい点を挙げます。

- **`imports`への宣言忘れ**: Componentと同じく、Directiveも使う側のComponentの`imports`に宣言する必要があります。付けたはずの振る舞いが効かないときは、まずここを確認します。
- **セレクターの角括弧忘れ**: `selector: 'appHighlight'`と角括弧なしで書くと、要素型セレクターと解釈され、属性として機能しません。属性Directiveでは`[appHighlight]`と角括弧で囲みます。
- **`ElementRef`の直接操作に頼りすぎる**: `nativeElement`を直接触ると、サーバーサイドレンダリング（第62章で扱います）などの環境で問題になることがあります。可能な範囲では`host`によるバインディングを優先します。

## まとめ

- 属性Directiveは`@Directive`で作り、`[appHighlight]`のような属性型セレクターを持ちます
- ホスト要素は`inject(ElementRef)`で取得でき、`nativeElement`でDOMを操作できます
- イベントやプロパティのバインディングは、`host`プロパティにまとめて宣言します
- 旧来の`@HostListener`・`@HostBinding`は、現在は`host`プロパティが標準です
- `input()`で外から値を受け取れ、Component間で共通する振る舞いを部品化できます

次章では、もうひとつの種類である構造Directiveと、その土台となる`ng-template`の仕組みを学びます。
