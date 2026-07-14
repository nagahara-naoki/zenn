---
title: "Directiveの実装とPipe"
---

この章では、Directiveを自分で作る方法と、値を整形するPipeを学びます。属性Directive・構造Directiveの実装を通して、テンプレートを拡張する力を身につけます。

:::message
**この章で学ぶこと**

- `@Directive`による属性Directiveの作り方
- ホスト要素の取得と操作
- 構造Directiveが何をするか
- `*`記法と`ng-template`の関係
- Pipeによる値の整形
- 代表的な組み込みPipe
:::

## 属性Directiveを作成する

前章で、Directiveには属性Directiveと構造Directiveがあり、属性Directiveは既存の要素に振る舞いを足すものだと学びました。この節では、その属性Directiveを実際に自分で作ります。

題材として、「マウスが乗ったら背景色を変える」ハイライトDirectiveを段階的に組み立てます。この過程で、ホスト要素の取得、イベントやプロパティのバインディング、そして外から値を受け取る`input()`の使い方を学びます。属性Directiveの作り方が分かると、Component間で共通する振る舞いを部品として切り出せるようになります。

### 属性Directiveの基本形

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

### ホスト要素を操作する

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

### hostでイベントとプロパティをバインドする

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

### 旧来のHostListener・HostBindingとの比較

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

### input()でDirectiveに値を渡す

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

### セレクターと命名規則

Directiveのセレクターには、いくつかの慣習があります。

- **接頭辞を付ける**: `app`のような接頭辞で、自作Directiveであることを示します。標準属性や他ライブラリとの衝突を防ぎます。
- **キャメルケースで書く**: 属性型セレクターは`appHighlight`のように、先頭を小文字にしたキャメルケースで書きます。HTMLの属性名として自然になります。
- **意味を表す名前にする**: `appHighlight`・`appAutoFocus`のように、振る舞いが分かる名前を付けます。

これらは、Componentの要素型セレクターがケバブケースだったのと対照的です。属性として付ける都合上、Directiveはキャメルケースが基本になります。

### よくあるつまずき

属性Directiveでつまずきやすい点を挙げます。

- **`imports`への宣言忘れ**: Componentと同じく、Directiveも使う側のComponentの`imports`に宣言する必要があります。付けたはずの振る舞いが効かないときは、まずここを確認します。
- **セレクターの角括弧忘れ**: `selector: 'appHighlight'`と角括弧なしで書くと、要素型セレクターと解釈され、属性として機能しません。属性Directiveでは`[appHighlight]`と角括弧で囲みます。
- **`ElementRef`の直接操作に頼りすぎる**: `nativeElement`を直接触ると、サーバーサイドレンダリング（第62章で扱います）などの環境で問題になることがあります。可能な範囲では`host`によるバインディングを優先します。

## 構造Directiveとng-templateの仕組み

前節では、既存の要素に振る舞いを足す属性Directiveを作りました。この節では、もうひとつの種類である構造Directiveを扱います。構造Directiveは、DOMの構造そのもの、つまり要素の追加や削除を担うDirectiveです。

第12章で学んだ`@if`・`@for`は、いまや制御フローの主役ですが、その前身である`*ngIf`・`*ngFor`は構造Directiveでした。この節では、構造Directiveがどのようにテンプレートを操作しているのか、その土台にある`ng-template`とあわせて解き明かします。仕組みを理解すると、テンプレートの一部を変数のように扱う高度な表現や、既存コードの読解ができるようになります。

### 構造Directiveとは

構造Directiveは、DOMに要素を追加したり、取り除いたりするDirectiveです。属性Directiveが「すでにある要素の見た目や振る舞いを変える」のに対し、構造Directiveは「要素そのものを出したり消したりする」点が異なります。

旧来の`*ngIf`を例に見てみます。

```html
<p *ngIf="isVisible()">条件が真のときだけ表示される段落</p>
```

`isVisible()`が`true`のときは`<p>`がDOMに現れ、`false`のときはDOMから消えます。表示・非表示をCSSで切り替えるのではなく、要素そのものをDOMに存在させたり、させなかったりするのが構造Directiveの特徴です。この先頭の`*`が、構造Directiveであることの目印です。

### *記法はng-templateの短縮形

この`*`という記法には、からくりがあります。`*`は、`ng-template`という要素を使った書き方の、短縮形（糖衣構文）なのです。先ほどの`*ngIf`は、内部的には次のように展開されます。

```html
<!-- *ngIf は、この形の短縮形 -->
<ng-template [ngIf]="isVisible()">
  <p>条件が真のときだけ表示される段落</p>
</ng-template>
```

`ng-template`は、「描画されずに保持されるテンプレートの断片」を表す要素です。中に書いた内容は、すぐには画面に出ません。構造Directiveが「いま表示すべき」と判断したときに、はじめてDOMへ差し込まれます。`*ngIf`は、この`ng-template`の中身を、条件に応じて出し入れしていたわけです。

`*`記法は、この`ng-template`の記述を省いて簡潔に書けるようにしたものです。1つの要素に付けられる構造Directiveは1つだけ、という制約があるのも、この展開の仕組みに由来します。

### ng-templateとは

`ng-template`は、それ単体では何も描画しません。あくまで「あとで使うためのテンプレートの入れ物」です。次のコードでは、`ng-template`の中身は画面に現れません。

```html
<ng-template>
  <p>この段落は、そのままでは表示されません</p>
</ng-template>
```

保持されたテンプレートを実際に画面へ出すには、それを差し込む指示が必要です。その指示を出すのが構造Directiveであり、指示の対象となるのが、次に説明する`TemplateRef`と`ViewContainerRef`です。

### TemplateRefとViewContainerRef

構造Directiveを自作するには、2つの部品を理解する必要があります。

- **`TemplateRef`**: `ng-template`が保持しているテンプレートの断片への参照です。「何を差し込むか」を表します。
- **`ViewContainerRef`**: テンプレートを差し込む先の、DOM上の入れ物です。「どこに差し込むか」を表します。

構造Directiveは、この2つを組み合わせて動きます。`ViewContainerRef`に対して「`TemplateRef`の中身を描画せよ」と命じれば要素が現れ、「消せ」と命じれば要素が消えます。`*ngIf`が行っていたのは、まさにこの操作でした。

### 自作の構造Directiveを作る

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

### ngTemplateOutletでテンプレートを差し込む

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

### 制御フロー時代の構造Directive

第12章で見たとおり、条件分岐や繰り返しという構造Directiveの代表的な用途は、いまや`@if`・`@for`という組み込み制御フローが担っています。そのため、`*ngIf`のような構造Directiveを日常的に書く場面や、自作する機会は、以前より大きく減りました。

とはいえ、`ng-template`と`ngTemplateOutlet`は現在も有効です。再利用可能なテンプレートの断片を扱う場面や、Componentに「差し込むテンプレート」を外から渡す設計では、いまも使われます。また、既存コードには`*ngIf`をはじめとする構造Directiveが数多く残っています。仕組みを理解しておくことは、そうしたコードを読み解くうえでも役立ちます。

:::message
新しく条件分岐や繰り返しを書くときは、構造Directiveではなく`@if`・`@for`を使います。構造Directiveの仕組みは、`ng-template`を用いた高度なテンプレート操作と、既存コードの読解のために押さえておく知識と位置づけてください。
:::

### よくあるつまずき

構造Directiveと`ng-template`まわりでつまずきやすい点を挙げます。

- **1つの要素に構造Directiveを2つ付ける**: `*`記法は`ng-template`への展開を伴うため、1要素に付けられる構造Directiveは1つだけです。旧来、`*ngIf`と`*ngFor`を同じ要素に付けられなかったのはこのためです。`ng-container`ではさんで階層を分けるか、組み込み制御フローを使います。
- **`ng-template`の中身が表示されない**: `ng-template`は、そのままでは描画されません。表示するには構造Directiveか`ngTemplateOutlet`による差し込みが必要です。「書いたのに出ない」と感じたら、この点を確認します。
- **`ng-container`と`ng-template`の混同**: `ng-container`は、余計なDOMを作らずに要素をまとめるための入れ物で、中身はそのまま描画されます。`ng-template`は、差し込まれるまで描画されない保持用の入れ物です。役割が異なる点に注意します。

## Pipeとテンプレートの再利用

第3部もこの節で最後です。ここまで、データバインディング・制御フロー・Directiveと、テンプレートを動かす道具を学んできました。最後に扱うのがPipe（パイプ）です。Pipeは、テンプレートの中で値を整形するための仕組みです。

日付を「2026年7月14日」の形で表示したい、数値を通貨の形式にしたい、文字列を大文字にしたい。こうした「表示のための変換」を、クラスを書き換えずにテンプレート側で完結できるのがPipeです。Angularには便利な組み込みPipeが多数用意されており、独自のPipeを作ることもできます。この節で、その使い方と仕組みを押さえます。

### Pipeとは

Pipeは、テンプレート内で値を変換する仕組みです。縦棒`|`の記号を使い、「値 | Pipe名」の形で書きます。この記号が、Unixのパイプのように「左の値を右へ流して加工する」イメージを表しています。

```html
<p>{{ name() | uppercase }}</p>
```

`name()`が`'yamada'`なら、`uppercase`というPipeを通って`'YAMADA'`と表示されます。元の`name`の値は変わりません。Pipeは、あくまで表示のための変換を行うだけです。クラス側のデータはそのままに、見せ方だけを整えられるのが特徴です。

### 代表的な組み込みPipe

Angularには、よく使う変換があらかじめPipeとして用意されています。代表的なものを挙げます。

| Pipe | 用途 | 例 |
|---|---|---|
| `date` | 日付の書式化 | `{{ today \| date }}` |
| `currency` | 通貨表示 | `{{ price \| currency }}` |
| `number` | 数値の桁区切り | `{{ count \| number }}` |
| `percent` | パーセント表示 | `{{ rate \| percent }}` |
| `uppercase` / `lowercase` | 大文字・小文字化 | `{{ name \| uppercase }}` |
| `json` | オブジェクトのJSON表示 | `{{ data \| json }}` |
| `async` | Observable・Promiseの購読 | `{{ data$ \| async }}` |

`json`は、デバッグ中にオブジェクトの中身を確認したいときに重宝します。`date`や`currency`は、地域（ロケール）に応じた書式で表示してくれます。これらの組み込みPipeは、いずれもimportなしでテンプレートに書けます。

### パラメーターとチェイン

Pipeには、コロン`:`でパラメーターを渡せます。たとえば`date`Pipeは、書式の文字列を受け取ります。

```html
<p>{{ today() | date:'yyyy年MM月dd日' }}</p>
```

これで「2026年07月14日」のような表示になります。パラメーターを複数渡すときは、コロンを重ねます。

```html
<p>{{ price() | currency:'JPY':'symbol' }}</p>
```

さらに、Pipeは連結（チェイン）できます。左から右へ順に適用されます。

```html
<p>{{ today() | date:'fullDate' | uppercase }}</p>
```

この例では、まず`date`で日付を書式化し、その結果を`uppercase`で大文字にしています。複数の変換を、テンプレート上で簡潔に組み合わせられます。

### 自作Pipeを作る

組み込みPipeにない変換は、自分で作れます。`@Pipe`デコレーターを付け、`PipeTransform`インターフェースの`transform`メソッドを実装します。Angular CLIで生成できます。

```bash
ng generate pipe truncate
```

例として、長い文字列を指定の長さで切り詰める`truncate`Pipeを作ります。

```ts:src/app/truncate-pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'truncate',
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit = 20, suffix = '...'): string {
    if (value.length <= limit) {
      return value;
    }
    return value.slice(0, limit) + suffix;
  }
}
```

`transform`の第1引数が、パイプの左側から流れてくる値です。第2引数以降が、コロンで渡すパラメーターに対応します。使うときは、Componentの`imports`に`TruncatePipe`を宣言したうえで、次のように書きます。

```html
<p>{{ description() | truncate:30 }}</p>
```

`description()`が30文字を超えると、30文字＋`...`に切り詰められます。パラメーターに既定値を持たせているため、`| truncate`と引数なしでも動きます。

なお、Pipeも第7章で学んだStandaloneの考え方が適用され、Angular 19以降は`standalone: true`の指定が不要です。単体でimportして使えます。

### Pure PipeとImpure Pipe

Pipeには、Pure（純粋）とImpure（非純粋）の2種類があります。既定はPureです。

Pure Pipeは、入力値が変わったときにだけ`transform`を実行します。正確には、入力の参照が変わったときです。同じ入力に対しては結果をそのまま使い回すため、効率的です。組み込みPipeの多くはPureで、通常はこれで十分です。

一方、配列やオブジェクトの「中身」が変わっても、参照が同じなら、Pure Pipeは再実行されません。中身の変化に追従したい場合は、`pure: false`を指定してImpure Pipeにします。

```ts:src/app/filter-pipe.ts
@Pipe({
  name: 'filterActive',
  pure: false,
})
export class FilterActivePipe implements PipeTransform {
  transform(items: Item[]): Item[] {
    return items.filter((item) => item.active);
  }
}
```

ただし、Impure Pipeは変更検知のたびに実行されるため、パフォーマンスに影響します。多用は避け、まずはデータ側を新しい参照として更新する（配列を作り直す）ことで、Pure Pipeのまま対応できないかを検討するのが望ましい書き方です。

### AsyncPipeで非同期の値を扱う

組み込みPipeの中でも特に重要なのが、`async`（AsyncPipe）です。これは、ObservableやPromiseから値を取り出し、テンプレートに表示します。

```html
<p>{{ user$ | async }}</p>
```

`user$`がObservableのとき、`async`Pipeは自動的にそれを購読（subscribe）し、流れてきた値を表示します。さらに、Componentが破棄されるときには購読を自動で解除してくれます。手動での購読と解除が不要になるため、メモリリークを防ぎつつ簡潔に書けます。

実務では、`async`Pipeを`@if`と組み合わせ、値が届いてから表示する書き方がよく使われます。第12章で学んだ`as`で、購読した値に名前を付けられます。

```html
@if (user$ | async; as user) {
  <p>{{ user.name }}</p>
} @else {
  <p>読み込み中</p>
}
```

こうすると、`user`が届くまでは「読み込み中」を表示し、届いたら中身を表示できます。`user$ | async`を何度も書かずに、`user`という名前で使い回せる点も利点です。ObservableやRxJSの詳しい仕組みは第8部で扱いますが、その値をテンプレートに映す入り口が、この`async`Pipeだと覚えておいてください。

### Pipeとメソッド呼び出しの違い

同じ変換は、クラスのメソッドを補間で呼んでも実現できます。それでもPipeを使う利点は、主に2つあります。

- **再利用性**: Pipeは、一度作れば複数のComponentで使い回せます。メソッドは、そのクラスの中に閉じてしまいます。
- **効率**: Pure Pipeは入力が変わったときだけ再計算します。テンプレートでメソッドを呼ぶと、変更検知のたびに実行される可能性があり、むだが生じやすくなります。

表示のための変換で、複数の場所で使う可能性があるものは、Pipeに切り出すのが適切です。逆に、そのComponentだけで一度きり使う単純な変換なら、`computed()`やメソッドでも構いません。

### Signalとの関係

モダンAngularでは、状態をSignalで持つことが増えています。Signalから導いた値の整形は、Pipeで行うことも、`computed()`で行うこともできます。使い分けの目安は、その変換が「表示のためのものか」「複数箇所で再利用するか」です。

`uppercase`や`date`のような汎用的な表示変換はPipeが向きます。一方、そのComponent固有の、状態から別の状態を導く計算は、`computed()`のほうが自然です。両者は競合するものではなく、役割に応じて併用します。Signalと`computed()`の詳細は、第6部で改めて扱います。

### よくあるつまずき

Pipeでつまずきやすい点を挙げておきます。

- **`imports`への宣言忘れ**: 自作Pipeや、一部の組み込みPipeは、使う側のComponentの`imports`に宣言が必要です。`{{ value | truncate }}`が効かないときは、まずここを確認します。
- **Impure Pipeの多用**: `pure: false`のPipeは変更検知のたびに実行され、パフォーマンスを損ないます。配列の絞り込みなどは、Pipeではなくデータ側（`computed()`など）で行うほうが安全な場面が多くあります。
- **Pipeで副作用を起こす**: `transform`は、値を変換して返すことに徹するべきです。中でデータを書き換えたり通信したりすると、予期しない再実行で問題が起きます。Pipeは純粋な変換だけを担う、と考えてください。

## まとめ

- 属性Directiveは`@Directive`で作り、`[appHighlight]`のような属性型セレクターを持ちます
- ホスト要素は`inject(ElementRef)`で取得でき、`nativeElement`でDOMを操作できます
- イベントやプロパティのバインディングは、`host`プロパティにまとめて宣言します
- 構造Directiveは、DOMに要素を追加・削除するDirectiveで、先頭の`*`が目印です
- `*`記法は`ng-template`を使った書き方の短縮形で、テンプレートの断片を出し入れします
- `TemplateRef`は差し込む内容、`ViewContainerRef`は差し込む先を表します
- Pipeは、テンプレート内で値を整形する仕組みで、縦棒`|`で書きます
- `date`・`currency`・`uppercase`・`json`・`async`など、便利な組み込みPipeが用意されています
- パラメーターはコロン`:`で渡し、複数のPipeは連結できます

次章からは、複数のComponentの間で状態をやり取りする方法に進みます。
