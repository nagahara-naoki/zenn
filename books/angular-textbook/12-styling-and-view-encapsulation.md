---
title: "第9章 スタイリングとView Encapsulation"
---

Componentには、テンプレートやクラスと並んで、スタイル（CSS）も付属します。この章では、Componentに書いたスタイルがどのように扱われるのか、なぜ他のComponentに影響しないのかを学びます。その鍵となるのが、View Encapsulation（ビューのカプセル化）という仕組みです。

CSSは本来、書いたスタイルがページ全体に及びます。あるComponentのために書いた`p { color: red; }`が、アプリケーション中のすべての`<p>`を赤くしてしまう、といったことが起こりえます。Angularは、この問題を既定で防いでくれます。その仕組みを理解しておくと、スタイルが「効かない」「漏れる」といったつまずきに、落ち着いて対処できます。

:::message
**この章で学ぶこと**

- Componentのスタイルがスコープされる仕組み
- View Encapsulationの3つのモード
- `:host`と`:host-context`によるホスト要素の指定
- グローバルスタイルとの使い分け
:::

## Componentのスタイルはスコープされる

Componentにスタイルを付けるには、`@Component`の`styleUrl`で別ファイルを指定するか、`styles`に直接書きます。

```ts:src/app/greeting.ts
@Component({
  selector: 'app-greeting',
  template: `<p>こんにちは</p>`,
  styleUrl: './greeting.css',
})
export class Greeting {}
```

```css:src/app/greeting.css
p {
  color: red;
}
```

この`p { color: red; }`は、`Greeting`のテンプレート内の`<p>`だけを赤くします。ほかのComponentにある`<p>`には影響しません。CSS本来のふるまいと違い、スタイルがComponentの中に閉じ込められているのです。この「スタイルがComponentの範囲に限定される」性質を、スコープと呼びます。

## View Encapsulationの仕組み

なぜスタイルがComponentの中に閉じるのでしょうか。その正体が、View Encapsulationです。

既定のモードでは、Angularはビルド時に、Componentの要素へ特別な属性（たとえば`_ngcontent-abc`のようなもの）を付け足します。そして、書いたCSSセレクターにも同じ属性を自動で付け加えます。結果として、`p { color: red; }`は内部的に「その属性を持つ`<p>`だけ」に限定され、他のComponentには届かなくなります。

開発者がこの属性を意識して書く必要はありません。Angularが自動で行うため、私たちはふつうにCSSを書くだけで、スコープされたスタイルを得られます。

実際に生成されるHTMLとCSSは、たとえば次のようなイメージです（属性名は自動で決まります）。

```html
<p _ngcontent-abc>こんにちは</p>
```

```css
p[_ngcontent-abc] {
  color: red;
}
```

`<p>`と、それに対応するCSSセレクターの両方に、同じ目印（`_ngcontent-abc`）が付いています。この目印が一致する要素にしかスタイルが当たらないため、他のComponentの`<p>`には影響しないのです。ブラウザの開発者ツールで要素を調べると、こうした属性が付いているのを実際に確認できます。

## 3つのカプセル化モード

View Encapsulationには、3つのモードがあります。`@Component`の`encapsulation`で指定します。

| モード | 挙動 |
|---|---|
| `Emulated`（既定） | 属性を付与してスコープを模倣する。他のComponentに漏れない |
| `None` | カプセル化しない。スタイルがアプリ全体に及ぶ |
| `ShadowDom` | ブラウザ標準のShadow DOMで、本物の分離を行う |

ふだんは、既定の`Emulated`のままで問題ありません。次のように書けば、明示的に指定することもできます。

```ts
import { Component, ViewEncapsulation } from '@angular/core';

@Component({
  selector: 'app-greeting',
  templateUrl: './greeting.html',
  styleUrl: './greeting.css',
  encapsulation: ViewEncapsulation.Emulated,
})
export class Greeting {}
```

`None`は、あえてスタイルを全体に効かせたいときに使いますが、他のComponentへ意図せず漏れる危険があります。`ShadowDom`は、ブラウザのShadow DOMを使って完全に分離しますが、外側からのスタイル調整が難しくなるなどの制約があります。いずれも、必要な理由があるときにだけ選ぶモードです。

たとえば`ShadowDom`は、Angularで作ったComponentを、Angular以外の環境でも使えるWeb Componentとして配布する、といった場面で選ばれます。逆に、アプリ内で完結する通常のComponentでは、既定の`Emulated`がもっとも扱いやすく、無理にモードを変える必要はありません。まずは既定のまま使い、明確な理由ができたときにだけ変更する、と考えておけば十分です。

## :hostでホスト要素を指定する

Componentのスタイルは、テンプレートの中身には効きますが、Component自身のタグ（ホスト要素）には、そのままでは効きません。たとえば`<app-greeting>`そのものに枠線を付けたいときは、`:host`という特別なセレクターを使います。

```css:src/app/greeting.css
:host {
  display: block;
  border: 1px solid #ccc;
  padding: 16px;
}
```

`:host`は、そのComponentのホスト要素自身を指します。Componentに余白や表示形式を与えたいときによく使います。

さらに、外側の状況に応じてスタイルを変えたいときは、`:host-context`を使います。たとえば、祖先に`dark`クラスがあるときだけ配色を変える、といった指定ができます。

```css
:host-context(.dark) {
  background: #222;
  color: #fff;
}
```

これは、アプリ全体のテーマ（明るい／暗い）に応じてComponentの見た目を切り替える、といった場面で役立ちます。

なお、`:host`は括弧を使って`:host(.active)`のように書くこともできます。これは「ホスト要素が`active`クラスを持つときだけ適用する」という条件付きの指定です。Component自身の状態に応じて、枠の色や背景を切り替えたいときに便利です。状態を表すクラスの付け外しは、第3部で学ぶクラスバインディングで行います。

## グローバルスタイルとの使い分け

Componentのスタイルがスコープされる一方で、アプリケーション全体に効かせたいスタイルもあります。フォントや配色の基本、リセットCSSなどです。これらは、プロジェクト直下の`src/styles.css`に書きます。ここに書いたスタイルは、カプセル化されず、アプリ全体に適用されます。

使い分けの目安は次のとおりです。

- **Componentのスタイル**: そのComponentだけに関わる見た目。大半のスタイルはこちら
- **グローバルスタイル（`styles.css`）**: アプリ全体で共通の土台となる見た目

迷ったら、まずComponentのスタイルに書き、本当に全体で共通のものだけをグローバルへ出す、と考えるとよいでしょう。

## ::ng-deepとその注意

子Componentの内部にまでスタイルを届かせたい、という要望が出ることがあります。このために`::ng-deep`という記法が使えますが、これはカプセル化の壁を越えてスタイルを漏らすもので、非推奨とされています。多用すると、どこからスタイルが効いているのかが追いにくくなり、カプセル化の利点を損ないます。

```css
/* 非推奨: カプセル化の壁を越えてしまう */
:host ::ng-deep .child-title {
  color: red;
}
```

子Componentの見た目を変えたい場合は、`::ng-deep`に頼るのではなく、子Component側にスタイルの入り口を用意するのが、より安全な方法です。まずはカプセル化を尊重する、という姿勢を基本にしてください。

## CSS変数でスタイルを受け渡す

子Componentの見た目を外から調整したいとき、`::ng-deep`に頼らない安全な方法のひとつが、CSSカスタムプロパティ（CSS変数）です。CSS変数は、カプセル化の壁を越えて受け渡せる性質を持っています。

子Component側では、変数を使ってスタイルを定義しておきます。第2引数に既定値を書けるので、外から指定がなくても破綻しません。

```css:src/app/badge.css
:host {
  background: var(--badge-bg, #eee); /* 既定は #eee */
}
```

使う側は、その変数に値を与えるだけで、見た目を変えられます。

```css
app-badge {
  --badge-bg: tomato;
}
```

この方法なら、子Componentが「どこを変更してよいか」を`--badge-bg`という形で明示できます。カプセル化を壊さずに、必要な部分だけを外から調整できるため、`::ng-deep`より見通しよく、安全にスタイルをカスタマイズできます。CSS変数は、テーマの切り替えにも便利です。アプリ全体で使う色を変数で定義しておけば、その変数の値を切り替えるだけで、配色を一括で変えられます。

## よくあるつまずき

スタイル周りでつまずきやすい点を挙げておきます。

- **書いたスタイルが効かない**: セレクターがテンプレートの構造と合っているかを確認します。Componentのスタイルは、そのテンプレート内の要素にしか当たりません。
- **子Componentの中身にスタイルが届かない**: これはView Encapsulationによる正しい挙動です。子側にCSS変数などの入り口を用意するか、子Component自身のスタイルとして書きます。
- **グローバルに書いたスタイルが効きすぎる**: `src/styles.css`はアプリ全体に及びます。特定のComponentだけに効かせたいスタイルは、そのComponent側に書きます。

## まとめ

- Componentのスタイルは既定でスコープされ、他のComponentに影響しません
- その仕組みがView Encapsulationで、既定の`Emulated`は属性を付与してスコープを模倣します
- モードには`Emulated`・`None`・`ShadowDom`があり、ふだんは既定のままで問題ありません
- ホスト要素自身は`:host`で、外側の状況に応じた指定は`:host-context`で行います
- 全体共通のスタイルは`src/styles.css`に書き、`::ng-deep`による越境は避けるのが安全です

次章では、ここまで学んだComponentを、どのように分割し、責務を配分すればよいのかという設計の視点を学びます。
