---
title: "テーマとスタイルシートの共有"
---

この章では、Shadow DOMのスタイルにまつわる2つの主題を扱います。

1つ目は、CSS Custom Propertiesによるテーマの公開です。色や余白など製品側が変えてよい値に名前を付け、利用者へ渡します。2つ目は、Constructable Stylesheetsによるスタイルの共有です。1つの`CSSStyleSheet`オブジェクトを作り、複数のShadow Rootで使い回します。

前者は「何を変えられるか」を決める設計の話、後者は「同じCSSをどう配るか」という実装の話です。

## 意味で名前を付けたCustom Propertiesを公開する

テーマを公開するとき、利用者が内部の全CSSを上書きできる必要はありません。変えてよい値だけに名前を付けます。このとき、内部実装のCSSプロパティ名をそのまま外へ出すと、用途が伝わりません。

```css
:host {
  --task-item-background-color: #fff;
  --task-item-border-color: #d7dce5;
  --task-item-accent-color: #2563eb;
  --task-item-radius: 0.75rem;
}

.container {
  color: inherit;
  background: var(--task-item-background-color);
  border: 1px solid var(--task-item-border-color);
  border-radius: var(--task-item-radius);
}

input {
  accent-color: var(--task-item-accent-color);
}
```

Custom Propertiesは継承するため、ホストへ設定した値がShadow Root内部まで届きます。利用側はホストのセレクターへ書くだけです。

```css
task-item {
  --task-item-background-color: #f5f3ff;
  --task-item-border-color: #c4b5fd;
  --task-item-accent-color: #7c3aed;
}
```

`purple`のような見た目の名前ではなく、`accent`のような役割の名前にすると、テーマを変更しても名前の意味が残ります。

## 既定値をコンポーネント側に持たせる

利用者がテーマ変数を設定しなくても、部品は読める状態で表示されるべきです。`var()`の第2引数にフォールバック値を書いておきます。

```css
.container {
  background: var(--task-item-background-color, Canvas);
  color: var(--task-item-text-color, CanvasText);
}
```

システムカラーの`Canvas`と`CanvasText`は、強制カラーモードにも馴染みやすい値です。固定色を使う場合も、コントラストを確認した既定テーマを用意します。

## 共通トークンと部品固有トークンを接続する

デザインシステム全体のトークンを内部から直接参照すると、部品が特定のページ設定へ強く依存します。部品固有の変数を経由させると、単体でも動きます。

```css
task-item {
  --task-item-background-color: var(--surface-color, #fff);
  --task-item-text-color: var(--text-color, #172033);
  --task-item-accent-color: var(--brand-color, #2563eb);
}
```

共通トークンがある環境ではその値を使い、単独利用では部品の既定値へ戻ります。

## Constructable Stylesheetで共有する

ここからが2つ目の主題です。各インスタンスへ同じ`<style>`を複製する代わりに、`CSSStyleSheet`をJavaScriptで作り、複数のShadow Rootへ採用できます。この方法で作るStylesheetをConstructable Stylesheetと呼びます。

```ts:src/task-item.styles.ts
export const taskItemStyles = new CSSStyleSheet();

taskItemStyles.replaceSync(`
  :host {
    display: block;
  }

  :host([hidden]) {
    display: none;
  }

  .container {
    display: flex;
    gap: 0.75rem;
    padding: 0.75rem;
    color: var(--task-item-text-color, #172033);
    background: var(--task-item-background-color, #fff);
    border: 1px solid var(--task-item-border-color, #d7dce5);
    border-radius: var(--task-item-radius, 0.75rem);
  }
`);
```

`replaceSync()`にCSSテキストを渡すと、その内容でルールが差し替わります。コンポーネント側では、Shadow Rootの`adoptedStyleSheets`へ追加します。

```ts
import { taskItemStyles } from './task-item.styles';

class TaskItem extends HTMLElement {
  constructor() {
    super();

    const root = this.attachShadow({ mode: 'open' });
    root.adoptedStyleSheets = [taskItemStyles];
  }
}
```

要素を100個作っても、パースされるCSSは1つです。同じインスタンスを共有するため、ルールを更新すると、採用しているすべてのShadow Rootへ即座に反映されます。

## style要素との使い分け

小さな単体部品なら、template内の`<style>`は読みやすく、追加のAPI知識も要りません。多くのインスタンスや複数部品で同じ基礎スタイルを共有する場合は、Constructable Stylesheetが向いています。

| 方法 | 向く場面 |
|---|---|
| `<style>` | 小さな部品、宣言的なtemplate、SSRでCSSをHTMLへ含めたい場合 |
| `adoptedStyleSheets` | 多数のインスタンス、共通Stylesheet、実行中の一括更新 |

Constructable Stylesheetは、同じDocumentで作成されたShadow Rootへ採用します。iframeなど別Documentへまたがる配布では、Documentごとに作る必要があります。

## 利用者の環境設定を尊重する

テーマはブランドカラーだけではありません。利用者のOS設定も反映の対象です。

```css
@media (prefers-reduced-motion: reduce) {
  .container {
    transition: none;
  }
}

@media (forced-colors: active) {
  .container {
    border-color: CanvasText;
  }
}
```

`color-scheme: light dark`をホストへ設定すれば、内部のフォーム部品やシステムカラーも配色へ追従します。

## まとめ

この章では、テーマの公開とスタイルシートの共有を学びました。

- テーマは、役割で命名したCSS Custom Propertiesとして公開します。値は継承によってホストから内部へ届きます。
- `var()`の第2引数へフォールバックを書き、利用者が何も設定しなくても読める既定値を用意します。
- 共通トークンは部品固有の変数を経由して参照すると、単体でも動きます。
- Constructable Stylesheetは、1つの`CSSStyleSheet`を`adoptedStyleSheets`で複数のShadow Rootへ共有する仕組みです。
- 小さな部品は`<style>`、多数のインスタンスや共通スタイルは`adoptedStyleSheets`を選びます。
- `prefers-reduced-motion`や`forced-colors`など、利用者のOS設定にも合わせます。

テーマAPIは見た目の公開契約です。次章では、スタイルを含む公開面全体を棚卸しし、コンポーネントの責務と変更規則を決めます。

## 参考資料

- [ShadowRoot.adoptedStyleSheets - MDN](https://developer.mozilla.org/en-US/docs/Web/API/ShadowRoot/adoptedStyleSheets)
- [Using shadow DOM: Applying styles - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM#applying_styles_inside_the_shadow_dom)
