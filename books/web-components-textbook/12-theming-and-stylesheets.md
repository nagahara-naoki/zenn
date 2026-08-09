---
title: "テーマは値の契約として渡し、Stylesheetを共有する"
---

コンポーネントのテーマを公開するとき、利用者が内部の全CSSを上書きできる必要はありません。色、余白、角丸など、製品側が変えてよい値に名前を付けて渡します。

## 意味で名前を付けたCustom Propertiesを公開する

内部実装のCSSプロパティ名をそのまま外へ出すと、用途が伝わりません。

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

利用側はホストへ値を設定します。

```css
task-item {
  --task-item-background-color: #f5f3ff;
  --task-item-border-color: #c4b5fd;
  --task-item-accent-color: #7c3aed;
}
```

`purple`のような見た目ではなく、`accent`のような役割で命名すると、テーマ変更後も名前の意味が残ります。

## 既定値をコンポーネント側に持たせる

利用者がテーマ変数を設定しなくても、部品は読める状態で表示されるべきです。`var()`の第2引数にもフォールバックを書けます。

```css
.container {
  background: var(--task-item-background-color, Canvas);
  color: var(--task-item-text-color, CanvasText);
}
```

システムカラーの`Canvas`と`CanvasText`は、強制カラーモードにも馴染みやすい値です。固定色を使う場合も、コントラストを確認した既定テーマを用意します。

## 共通トークンと部品固有トークンを接続する

デザインシステム全体のトークンを直接内部で参照すると、部品が特定のページ設定へ強く依存します。部品固有の変数に接続すると、単体でも動きます。

```css
task-item {
  --task-item-background-color: var(--surface-color, #fff);
  --task-item-text-color: var(--text-color, #172033);
  --task-item-accent-color: var(--brand-color, #2563eb);
}
```

共通トークンが存在する環境ではその値を使い、単独利用では部品の既定値へ戻ります。

## Constructable Stylesheetを複数の要素で共有する

各インスタンスへ同じ`<style>`を複製する代わりに、`CSSStyleSheet`を作って複数のShadow Rootへ採用できます。この方法で作るStylesheetをConstructable Stylesheetと呼びます。

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

コンポーネントでは`adoptedStyleSheets`へ追加します。

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

同じ`CSSStyleSheet`インスタンスを共有するため、ルールを更新すると採用しているすべてのShadow Rootへ反映されます。

## style要素との使い分け

小さな単体部品なら、template内の`<style>`は読みやすく、追加のAPI知識も要りません。多くのインスタンスや複数部品で同じ基礎スタイルを共有する場合は、Constructable Stylesheetが向いています。

| 方法 | 向く場面 |
|---|---|
| `<style>` | 小さな部品、宣言的なtemplate、SSRでCSSをHTMLへ含めたい場合 |
| `adoptedStyleSheets` | 多数のインスタンス、共通Stylesheet、実行中の一括更新 |

Constructable Stylesheetは同じDocumentで作成されたShadow Rootへ採用します。iframeなど別Documentへまたがる配布では、Documentごとに作る必要があります。

## 利用者の環境設定を上書きしない

テーマはブランドカラーだけではありません。利用者のOS設定も尊重します。

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

`color-scheme: light dark`をホストへ設定すれば、内部のフォーム部品やシステムカラーも配色へ追従できます。

テーマAPIは見た目の公開契約です。次章では、スタイルを含む公開面全体を棚卸しし、コンポーネントの責務と変更規則を決めます。

## 参考資料

- [ShadowRoot.adoptedStyleSheets - MDN](https://developer.mozilla.org/en-US/docs/Web/API/ShadowRoot/adoptedStyleSheets)
- [Using shadow DOM: Applying styles - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM#applying_styles_inside_the_shadow_dom)
