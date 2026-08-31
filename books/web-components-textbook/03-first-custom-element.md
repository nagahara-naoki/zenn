---
title: "はじめてのCustom Element"
---

最初に作るのは、`name`属性で受け取った名前を表示する`<hello-card>`です。Shadow DOMはまだ使いません。独自要素の定義、登録、利用という最短経路だけを確かめます。

Custom Elementの登録は、ブラウザが持つ要素名の対応表へ新しい項目を追加する操作です。名前だけを登録するのではなく、その名前の要素を生成したときに使うクラスを結びつけます。以後、HTML Parser、`document.createElement()`、`querySelector()`が扱う同じDOM要素に、そのクラスのプロパティとメソッドが加わります。

この章では表示を作り込まず、次の3段階だけを確認します。

1. `HTMLElement`を継承したクラスに動作を書く
2. 要素名とクラスを`customElements.define()`で登録する
3. 通常のHTMLタグとして利用する

これは以降のすべての章の土台です。テンプレートやShadow DOMを使っても、独自要素がブラウザへ登録される仕組み自体は変わりません。

## Viteで実行環境を作る

空のディレクトリで始めるより、Viteの`vanilla-ts`テンプレートを使うと開発サーバーとTypeScriptの設定が一度にそろいます。

```sh
pnpm create vite web-components-lab --template vanilla-ts
cd web-components-lab
pnpm install
pnpm dev
```

表示されたURLをブラウザで開いてください。以降は、生成された`src/main.ts`と`index.html`を書き換えます。

:::message
Viteの生成内容はバージョンによって変わります。本書では`index.html`から`src/main.ts`をES Moduleとして読み込めれば進められます。
:::

## HTMLElementを継承する

ブラウザ上の通常のHTML要素は`HTMLElement`を継承しています。独自要素も同じクラス階層へ参加させます。

```ts:src/hello-card.ts
export class HelloCard extends HTMLElement {
  connectedCallback(): void {
    const name = this.getAttribute('name') ?? 'ゲスト';
    this.textContent = `こんにちは、${name}さん`;
  }
}
```

`connectedCallback()`は、要素が文書へ接続されたときにブラウザから呼ばれるメソッドです。ここでは`name`属性を読み、`textContent`に安全に表示しています。

クラスを書いただけでは、`<hello-card>`という名前との関係をブラウザは知りません。`customElements.define()`で登録します。

```ts:src/main.ts
import { HelloCard } from './hello-card';

customElements.define('hello-card', HelloCard);
```

HTMLから利用してみましょう。

```html:index.html
<!doctype html>
<html lang="ja">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Web Components Lab</title>
    <script type="module" src="/src/main.ts"></script>
  </head>
  <body>
    <hello-card name="佐藤"></hello-card>
  </body>
</html>
```

画面に「こんにちは、佐藤さん」と表示されれば成功です。

## 要素名に必要なハイフン

Autonomous Custom Elementの名前には`hello-card`のようなハイフンが必要です。将来HTML標準に同名の要素が追加されても衝突しないように、組み込み要素と名前空間を分ける役割があります。

製品用のコンポーネントでは、`acme-button`や`shop-product-card`のような固有の接頭辞を決めると衝突を減らせます。本書のサンプルでは役割を読み取りやすくするため、`task-`を接頭辞として使います。

次の名前は登録できません。

```ts
customElements.define('card', HelloCard);
// DOMException: 名前にハイフンがない
```

## 通常のDOM APIで操作できる

登録した要素は特別なオブジェクトではありません。`querySelector()`で取得し、属性を変更できます。

```ts
const card = document.querySelector('hello-card');

if (card instanceof HelloCard) {
  console.log(card.getAttribute('name'));
}
```

この時点では、登録後に`name`属性を変えても表示は更新されません。属性変更を監視する方法は『属性とプロパティ』の章で扱います。

## 外部の文字列はtextContentで表示する

名前を表示するだけなら`innerHTML`は不要です。

```ts
this.innerHTML = `こんにちは、${name}さん`;
```

`name`に外部入力が含まれると、このコードは文字列をHTMLとして解釈します。最初の例で`textContent`を使ったのは、文字列を文字列のまま扱うためです。HTMLを組み立てる方法とセキュリティは『セキュリティとXSS対策』の章で改めて整理します。

## 開発者ツールで定義を確かめる

ブラウザのコンソールで登録状態を確認できます。

```js
customElements.get('hello-card');
```

`HelloCard`クラスが返れば登録済みです。まだ登録されていない名前では`undefined`が返ります。

```js
document.querySelector('hello-card') instanceof HTMLElement;
// true
```

Custom ElementはHTML要素そのものです。この一点が、以降の属性、イベント、フォーム、アクセシビリティの土台になります。

## 演習

`greeting`属性を追加し、次のHTMLが「こんばんは、佐藤さん」と表示されるようにしてください。

```html
<hello-card name="佐藤" greeting="こんばんは"></hello-card>
```

属性が省略された場合は「こんにちは」を使います。

## まとめ

この章では、最小のCustom Elementを作って登録しました。

- Custom Elementは`HTMLElement`を継承したクラスとして書きます。
- `customElements.define()`が要素名とクラスを結びつけ、以後そのタグから生成される要素にクラスの振る舞いが加わります。
- `connectedCallback()`は要素が文書へ接続されたときに呼ばれます。
- Autonomous Custom Elementの名前にはハイフンが必要です。
- 外部から受け取った文字列は`textContent`で表示します。
- 登録した要素は`querySelector()`などの通常のDOM APIで操作できます。

次章では、`define()`とアップグレードの仕組みを詳しく見ていきます。

## 参考資料

- [Getting Started - Vite](https://vite.dev/guide/)
- [Using custom elements - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements)
