---
title: "Litとは何か"
---

Litとは、Web Components標準の上に薄く乗るライブラリで、宣言的テンプレートと差分更新を補うものです。ネイティブのWeb Components APIは、要素の登録、Shadow DOM、属性、イベント、Slotを提供します。状態が変わったときのDOM差分更新や宣言的なtemplateは提供しません。Litが引き受けるのはその部分です。

`LitElement`は`HTMLElement`を継承しています。完成した部品の公開面はCustom Elementのままで、利用側から見れば標準のタグです。

## Litを追加する

```sh
pnpm add lit
```

最初はデコレーターを使わず、通常のclass fieldと`static properties`で仕組みを確認します。

```ts:src/lit-task-item.ts
import { LitElement, css, html } from 'lit';
import type { PropertyValues } from 'lit';
import type { Task, TaskToggleDetail } from './task-item.types.js';

export class LitTaskItem extends LitElement {
  static properties = {
    task: { attribute: false },
    completed: {
      type: Boolean,
      reflect: true,
    },
  };

  static styles = css`
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
      border: 1px solid var(--task-item-border-color, #d7dce5);
      border-radius: var(--task-item-radius, 0.75rem);
    }

    :host([completed]) .label {
      color: #667085;
      text-decoration: line-through;
    }
  `;

  declare task?: Task;
  declare completed: boolean;

  constructor() {
    super();
    this.completed = false;
  }

  render() {
    return html`
      <article class="container" part="container">
        <input
          type="checkbox"
          .checked=${this.completed}
          @change=${this.#handleChange}
        >
        <span class="label"><slot name="label"></slot></span>
        <slot name="meta"></slot>
      </article>
    `;
  }

  #handleChange(event: Event): void {
    if (!(event.currentTarget instanceof HTMLInputElement)) return;
    if (!this.task) return;

    const detail: TaskToggleDetail = {
      id: this.task.id,
      completed: event.currentTarget.checked,
    };

    this.dispatchEvent(
      new CustomEvent<TaskToggleDetail>('task-toggle', {
        bubbles: true,
        composed: true,
        detail,
      }),
    );
  }
}

customElements.define('lit-task-item', LitTaskItem);
```

登録はいつもどおり`customElements.define()`です。Shadow DOMの生成やstyleの適用は`LitElement`が済ませます。

## reactive propertyが更新を予約する

`static properties`へ宣言した値をLitではreactive propertyと呼びます。値が変わると更新が予約され、同じ処理内の複数変更はまとめられます。

```ts
item.task = task;
item.completed = task.completed;

await item.updateComplete;
```

`task`は`attribute: false`なのでプロパティだけで受け取ります。`completed`はBoolean属性と対応し、`reflect: true`でプロパティ変更を属性へ反映します。『属性とプロパティ』の章で手作業した変換とreflectionを、Litが管理しています。

## templateのbindingは種類を記号で分ける

Lit templateでは、値を渡す場所によって記法が変わります。

```ts
html`
  <task-item
    .task=${task}
    ?completed=${task.completed}
    priority=${task.priority}
    @task-toggle=${handleToggle}
  ></task-item>
`
```

| 記法 | 渡す先 |
|---|---|
| `${value}` | テキストまたは通常属性 |
| `.task=${value}` | DOMプロパティ |
| `?completed=${value}` | 真偽属性 |
| `@event=${handler}` | イベントリスナー |

プロパティと属性の違いが記法に現れるため、オブジェクトを誤って文字列属性へ渡す事故を減らせます。

## Litのライフサイクルと標準の対応

LitElementにも標準のCustom Elementライフサイクルがあります。overrideする場合は`super`を呼びます。

```ts
connectedCallback(): void {
  super.connectedCallback();
  window.addEventListener('online', this.#handleOnline);
}

disconnectedCallback(): void {
  window.removeEventListener('online', this.#handleOnline);
  super.disconnectedCallback();
}
```

Lit固有の更新タイミングもあります。

| メソッド・値 | 用途 |
|---|---|
| `willUpdate()` | 描画前に派生値を計算する |
| `render()` | templateを返す |
| `firstUpdated()` | 最初の描画後に一度だけ処理する |
| `updated()` | 描画後に変更プロパティへ応答する |
| `updateComplete` | 更新完了を待つPromise |

`updated()`で毎回プロパティを変更すると、更新ループを作ることがあります。派生値は`render()`内で計算するか、`willUpdate()`で変更条件を確認します。

## SlotとCSS Partsはそのまま使う

Litは独自のSlotやスタイルAPIへ置き換えません。template内の`<slot>`、`part`、CSS Custom Propertiesはブラウザ標準です。

```ts
render() {
  return html`
    <article part="container">
      <slot name="label"></slot>
    </article>
  `;
}
```

利用側のHTML構造はネイティブ実装と同じです。ここでは比較用に`<lit-task-item>`という別名で登録しています。実際の移行では、登録するクラスだけを差し替え、公開タグ名`<task-item>`は変えません。

```html
<lit-task-item>
  <span slot="label">原稿をレビューする</span>
</lit-task-item>
```

## デコレーターはチームのTypeScript設定と合わせる

Litは`@customElement`、`@property`、`@state`などのデコレーターを提供します。

```ts
@customElement('task-item')
export class TaskItem extends LitElement {
  @property({ attribute: false })
  accessor task?: Task;
}
```

JavaScriptの標準デコレーターとTypeScriptの旧来のexperimental decoratorsでは、設定と出力が異なります。ライブラリのコンパイル方針を決め、利用者へ必要な設定を押し付けないようにします。デコレーターを使わない`static properties`も正式な選択肢です。

## Litを選ぶ境界

表示が小さく更新箇所も2、3個なら、ネイティブDOM APIのほうが依存も学習項目も少なく済みます。条件分岐、一覧、非同期状態、複数の派生表示が増えたら、Litの宣言的templateが読みやすさを保ちます。

Litへ移行しても、属性、プロパティ、イベント、Slot、Partsの公開契約は変えません。この原則を守れば、内部実装を段階的に置き換えられます。

## まとめ

この章では、Litの位置づけと基本的な書き方を学びました。

- Litは、標準のWeb Componentsに宣言的テンプレートと差分更新を足す薄いライブラリです。
- `LitElement`は`HTMLElement`を継承するため、公開面はCustom Elementのままです。
- `static properties`へ宣言したreactive propertyは、値の変更で更新を予約し、複数の変更をまとめます。
- `reflect: true`で属性への反映をLitが管理します。
- templateのbindingは、`.`がDOMプロパティ、`?`が真偽属性、`@`がイベントリスナーと記号で分かれます。
- 標準のライフサイクルはそのまま使え、overrideするときは`super`を呼びます。
- Slot、CSS Parts、CSS Custom Propertiesはブラウザ標準のまま利用します。

次章の『タスク管理UIを完成させる』では、ネイティブ実装で学んだ4つの要素を1つのタスク管理UIへ統合します。

## 参考資料

- [What is Lit?](https://lit.dev/docs/)
- [Reactive properties - Lit](https://lit.dev/docs/components/properties/)
- [Components overview - Lit](https://lit.dev/docs/components/overview/)
