---
title: "要素の定義とアップグレード"
---

アップグレードとは、すでにDOM上にある要素を、あとから登録されたCustom Elementのクラスへ結びつける処理です。ブラウザは未知のタグを見つけても捨てず、いったん通常の要素としてDOMに置きます。その後`customElements.define()`が呼ばれた時点で、文書中の該当する要素をまとめてそのクラスのインスタンスに変えます。

この遅延があるため、Custom ElementはHTMLを先に配信し、JavaScriptを後から読み込む構成でも使えます。この章では、定義の登録方法と、アップグレードが設計に及ぼす影響を扱います。

## 登録前の要素はHTMLElementとして扱われる

ハイフンを含む有効なCustom Element名は、定義前でも`HTMLUnknownElement`にはならず、`HTMLElement`として扱われます。

```html
<task-item>仕様書を読む</task-item>
```

```js
const item = document.querySelector('task-item');

console.log(item instanceof HTMLElement); // true
console.log(customElements.get('task-item')); // undefined
```

ここで定義を登録します。

```ts
class TaskItem extends HTMLElement {
  connectedCallback(): void {
    this.dataset.upgraded = 'true';
  }
}

customElements.define('task-item', TaskItem);
```

登録時点で文書に存在する`<task-item>`も`TaskItem`へアップグレードされ、`connectedCallback()`が実行されます。

```js
console.log(item instanceof TaskItem); // true
console.log(item.dataset.upgraded); // "true"
```

## defineは一度しか呼べない

グローバルな`CustomElementRegistry`は`window.customElements`から参照します。主なメソッドは3つです。

| メソッド | 用途 |
|---|---|
| `define(name, constructor)` | 名前とクラスを登録する |
| `get(name)` | 登録済みのクラスを取得する |
| `whenDefined(name)` | 登録が終わるまで待つ |

同じ名前を2回登録すると`DOMException`が発生します。開発中のHot Module Replacementや、複数の入口から同じモジュールを読み込む構成では注意が必要です。

```ts
export function registerTaskItem(): void {
  if (!customElements.get('task-item')) {
    customElements.define('task-item', TaskItem);
  }
}
```

ライブラリでは、クラスのexportと登録処理を分ける方法もあります。

```ts:src/task-item.ts
export class TaskItem extends HTMLElement {}
```

```ts:src/register.ts
import { TaskItem } from './task-item';

export function registerTaskItem(
  name = 'task-item',
): void {
  if (!customElements.get(name)) {
    customElements.define(name, TaskItem);
  }
}
```

利用者は登録名を管理でき、ライブラリをimportしただけでグローバル状態が変わることも避けられます。『npmパッケージ化とドキュメント』の章では、この分け方をパッケージの`exports`へ反映します。

## whenDefinedで登録完了を待つ

要素を操作するコードが定義モジュールとは別に読み込まれる場合、`whenDefined()`で順序を合わせられます。

```ts
await customElements.whenDefined('task-item');

const item = document.querySelector('task-item');
if (item instanceof TaskItem) {
  item.completed = true;
}
```

ページ全体を`DOMContentLoaded`まで待つ必要はありません。必要な要素の定義だけを待てます。

## コンストラクターに書いてよい処理

Custom Elementのコンストラクターには、ブラウザが要素を生成する途中で呼ぶという制約があります。ここでは次の作業に絞ります。

- `super()`を最初に呼ぶ
- Shadow Rootを作る
- 内部状態を初期化する
- 自分自身に必要な内部DOMを用意する

属性やLight DOMの子要素に依存する処理、文書全体への問い合わせ、外部通信は`connectedCallback()`以降へ回します。

```ts
class TaskItem extends HTMLElement {
  #root: ShadowRoot;

  constructor() {
    super();
    this.#root = this.attachShadow({ mode: 'open' });
  }
}
```

## Autonomous Custom Elementを基本にする

Custom Elementには、`HTMLElement`を直接継承するAutonomous Custom Elementと、`HTMLButtonElement`などを拡張するCustomized Built-in Elementがあります。

```ts
class ConfirmButton extends HTMLButtonElement {}

customElements.define('confirm-button', ConfirmButton, {
  extends: 'button',
});
```

利用時は`is`属性を使います。

```html
<button is="confirm-button">保存</button>
```

組み込み要素の意味を継承できる利点はありますが、配布先の対応状況とフレームワーク連携を確認する必要があります。本書の通しサンプルでは、扱いやすいAutonomous Custom Elementを基本にします。アクセシビリティはShadow DOM内部で`button`や`input`を再利用して守ります。

## 登録とインスタンス生成を混同しない

`customElements.define()`はインスタンスを作りません。クラスと名前を登録するだけです。インスタンスはHTMLパーサーや`document.createElement()`が作ります。

```ts
const item = document.createElement('task-item');
item.textContent = 'テストを書く';
document.body.append(item);
```

ブラウザが登録済みの名前を見て`TaskItem`を生成し、文書へ追加された時点で`connectedCallback()`を呼びます。

## まとめ

この章では、Custom Elementの定義とアップグレードを扱いました。

- アップグレードは、定義前からDOMにあった要素を登録後にクラスのインスタンスへ変える処理です。
- 有効な名前の要素は定義前でも`HTMLElement`として存在し、属性とLight DOMを保持します。
- `define()`は同じ名前で二度呼べないため、登録処理を関数に切り出して`get()`で確認します。
- 別モジュールから要素を操作する場合は`whenDefined()`で登録完了を待てます。
- コンストラクターでは`super()`、Shadow Root、内部状態など自分自身の土台だけを用意します。
- 本書の通しサンプルは、扱いやすいAutonomous Custom Elementを基本にします。

次章では、この接続と切断を含むライフサイクルを扱います。

## 参考資料

- [Custom elements - HTML Living Standard](https://html.spec.whatwg.org/multipage/custom-elements.html)
- [CustomElementRegistry - MDN](https://developer.mozilla.org/en-US/docs/Web/API/CustomElementRegistry)
