---
title: "Templateで内部を複製し、Slotで利用者の内容を受け入れる"
---

同じ内部DOMを要素のインスタンスごとに作るなら、`<template>`にひな形を置いて複製できます。表示する文字や補助情報を利用者に書いてもらう部分には`<slot>`を置きます。

この2つは似た「HTMLを組み立てる機能」に見えますが、所有者が違います。テンプレートはコンポーネント作者が管理する内部構造です。Slotへ渡す内容は利用者がLight DOMに書き、コンポーネントは表示位置だけを用意します。

たとえばカードの枠、チェックボックス、余白は内部テンプレートに置けます。一方、タスク名に`strong`やリンクを含めてよいなら、その内容はSlotで受け取ります。内部へコピーするのではなく、利用者が所有するノードを表示上の位置へ割り当てる点が重要です。

## templateの内容は読み込み時に表示されない

`<template>`の子要素は、通常のDOMとして画面に表示されません。内容は`HTMLTemplateElement.content`に`DocumentFragment`として保持されます。

```html
<template id="task-item-template">
  <style>
    :host {
      display: block;
    }

    label {
      display: flex;
      gap: 0.5rem;
      align-items: center;
    }
  </style>

  <label>
    <input type="checkbox">
    <span><slot></slot></span>
  </label>
</template>
```

Custom Elementのコンストラクターで内容を複製します。

```ts
const template = document.querySelector<HTMLTemplateElement>(
  '#task-item-template',
);

class TaskItem extends HTMLElement {
  constructor() {
    super();

    const root = this.attachShadow({ mode: 'open' });
    if (!template) throw new Error('templateが見つかりません');

    root.append(template.content.cloneNode(true));
  }
}
```

`cloneNode(true)`の`true`は、子孫を含めて複製する指定です。ひとつの`DocumentFragment`をそのまま複数回appendすると、最初の場所から次の場所へ移動してしまいます。各インスタンスで複製してください。

## モジュール内templateなら部品と一緒に配布できる

HTMLファイルへtemplateを置くと、コンポーネント利用者がそのひな形まで準備しなければなりません。配布する部品では、モジュール内でtemplateを作る方法が扱いやすくなります。

```ts
const template = document.createElement('template');
template.innerHTML = `
  <style>
    :host { display: block; }
    label { display: flex; gap: 0.5rem; }
  </style>
  <label>
    <input type="checkbox">
    <span><slot></slot></span>
  </label>
`;
```

このHTML文字列は作者がソースコードへ固定した信頼済みの文字列です。利用者の入力を埋め込んでいません。外部データはSlot、`textContent`、DOMプロパティで渡します。

## デフォルトSlotは名前のない子を受け取る

名前のない`<slot>`には、`slot`属性を持たないLight DOMのノードが割り当てられます。

```html
<task-item>原稿をレビューする</task-item>
```

文字列「原稿をレビューする」はLight DOMに残ったまま、Shadow Treeの`<slot>`位置へ描画されます。`task-item.textContent`からも読めます。

## 名前付きSlotで役割を分ける

タスク名と期限を別々に受け取りたいなら、Slotへ名前を付けます。

```html
<template id="task-item-template">
  <article>
    <label>
      <input type="checkbox">
      <slot name="label">名称未設定</slot>
    </label>
    <small><slot name="meta"></slot></small>
  </article>
</template>
```

利用者は子要素の`slot`属性で割り当て先を指定します。

```html
<task-item>
  <span slot="label">原稿をレビューする</span>
  <time slot="meta" datetime="2026-08-12">8月12日まで</time>
</task-item>
```

`<slot>`自身の子に書いた「名称未設定」は、割り当てられるノードがない場合のフォールバックです。必須の表示が欠けても、空白の部品にならずに済みます。

## slotchangeで割り当ての変化を知る

利用者が子要素を追加・削除すると、対象Slotで`slotchange`イベントが発生します。

```ts
const labelSlot = root.querySelector<HTMLSlotElement>(
  'slot[name="label"]',
);

labelSlot?.addEventListener('slotchange', () => {
  const elements = labelSlot.assignedElements({ flatten: true });
  console.log(elements);
});
```

`assignedElements()`は割り当てられた要素だけを返します。テキストノードも必要なら`assignedNodes()`を使います。

Slotへ割り当てられた要素の内部で文字列だけが変わっても、割り当て関係は変わらないため`slotchange`は発生しません。内容変更まで監視する必要があるなら、対象ノードへ`MutationObserver`を設定します。

## Slotは利用者へHTMLの表現力を残す

文字列プロパティだけでラベルを受け取ると、利用者は`<strong>`、`<time>`、アイコン、言語属性などを追加できません。

```ts
item.label = '原稿をレビューする';
```

Slotなら、意味のあるHTMLをそのまま渡せます。

```html
<task-item>
  <span slot="label">
    <strong>原稿</strong>をレビューする
  </span>
</task-item>
```

構造化された表示内容はSlot、プログラムが扱うデータはプロパティという分担が有効です。ただし、名前付きSlotを無制限に増やすと内部レイアウトを守れません。名前付きSlotは、利用者に許可する構成上の拡張点として設計します。

次章では、Shadow DOM内部の操作を外へ伝えるイベントを設計します。

## 参考資料

- [Using templates and slots - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_templates_and_slots)
- [`<slot>` - HTML Living Standard](https://html.spec.whatwg.org/multipage/scripting.html#the-slot-element)
