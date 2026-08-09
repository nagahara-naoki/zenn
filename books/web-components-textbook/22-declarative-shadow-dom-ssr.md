---
title: "Declarative Shadow DOMでJavaScript実行前に内部を表示する"
---

`attachShadow()`はJavaScriptが実行されるまで内部DOMを作れません。サーバーから返すHTMLにShadow Rootの内容を含めるDeclarative Shadow DOMを使うと、HTMLパーサーが初期表示を組み立てます。

## templateのshadowrootmodeでShadow Rootを宣言する

サーバーは次のHTMLを返します。

```html
<task-item completed>
  <template shadowrootmode="open">
    <style>
      :host { display: block; }
      .container {
        display: flex;
        gap: 0.75rem;
        padding: 0.75rem;
      }
    </style>

    <label class="container">
      <input type="checkbox" checked>
      <slot name="label">名称未設定</slot>
    </label>
  </template>

  <span slot="label">原稿をレビューする</span>
</task-item>
```

ブラウザは`template`を通常の要素として残さず、その親である`<task-item>`へShadow Rootを取り付けます。JavaScriptが遅れても、スタイルと内容を含む初期画面が表示されます。

## クライアントは既存のShadow Rootを再利用する

Custom Elementのコンストラクターで無条件に`attachShadow()`すると、サーバーが作った既存のShadow Rootと衝突します。先に`this.shadowRoot`を確認します。

```ts
class TaskItem extends HTMLElement {
  #root: ShadowRoot;
  #checkbox: HTMLInputElement;

  constructor() {
    super();

    this.#root = this.shadowRoot ?? this.attachShadow({ mode: 'open' });

    const checkbox = this.#root.querySelector('input');
    if (!(checkbox instanceof HTMLInputElement)) {
      this.#root.append(template.content.cloneNode(true));
    }

    const resolved = this.#root.querySelector('input');
    if (!(resolved instanceof HTMLInputElement)) {
      throw new Error('task-itemのinputを初期化できません');
    }

    this.#checkbox = resolved;
  }
}
```

サーバー描画があれば既存ノードへイベントを接続し、なければクライアント用templateから作ります。

:::message alert
同じmodeで既存のDeclarative Shadow Rootへ`attachShadow()`を呼ぶと、ブラウザは既存Rootを返しますが内容をクリアします。SSR内容を再利用する実装では、先に`this.shadowRoot`を確認してください。
:::

## サーバーとクライアントで同じ公開状態を使う

サーバーが`completed`属性と`checked`の表示を別々に計算すると、片方だけ変わる恐れがあります。公開状態をHTML属性として残し、サーバーとクライアントが同じ規則から描画します。

```ts
get completed(): boolean {
  return this.hasAttribute('completed');
}

#render(): void {
  this.#checkbox.checked = this.completed;
}
```

クライアントがアップグレードした直後に同じ状態を描画しても、見た目は変わりません。このサーバー出力とクライアント初期状態の一致が、安定した引き継ぎの条件です。

## SSRとhydrationを分けて考える

Declarative Shadow DOMでHTMLを表示するだけなら、サーバーレンダリングです。既存ノードへイベントと状態更新を接続し、作り直さず再利用する処理をhydrationと呼びます。

ネイティブCustom Elementでは、必要なノードを取得してリスナーを登録する処理を自分で書きます。Litなどのテンプレートライブラリでは専用のSSR・hydration機能が提供されることがありますが、利用するパッケージの安定性と制約を確認してください。

## shadowrootdelegatesfocusもHTMLから指定できる

フォーカス委譲が必要なら、Declarative Shadow DOM側にも指定します。

```html
<template
  shadowrootmode="open"
  shadowrootdelegatesfocus
>
  <!-- ... -->
</template>
```

クライアントだけ`delegatesFocus: true`にしても、初期Shadow Rootの設定とは一致しません。Shadow Root作成時に決まる値はサーバーとクライアントでそろえます。

## ストリーミングHTMLと相性がよい

Declarative Shadow DOMはHTMLパーサーが処理するため、レスポンスを上から受け取りながらShadow Rootを構築できます。クライアントJavaScriptが大きいページでも、部品の枠と静的内容を先に表示できます。

ただし、`innerHTML`へ後から同じ文字列を代入すれば常にDeclarative Shadow DOMになるとは限りません。主な用途は、文書のHTMLレスポンスとしてパースさせることです。動的なHTML挿入では、利用するパースAPIがDeclarative Shadow Rootを許可するか確認します。

## 未対応環境ではLight DOMをフォールバックにする

対象ブラウザが`shadowrootmode`へ対応しているかは、リリース前に互換性表と実機で確認します。対応しない環境でも重要な内容を読めるように、タスク名などの意味情報はLight DOMへ残します。

```html
<task-item>
  <template shadowrootmode="open"><!-- 強化されたUI --></template>
  <span slot="label">原稿をレビューする</span>
</task-item>
```

SSRはJavaScriptをなくす仕組みではありません。初期表示をHTMLへ戻し、その後の操作だけをJavaScriptへ引き継ぐ設計です。

新しいWeb Components APIを製品へ入れる判断基準は、第23章でCustom StatesとScoped Custom Element Registriesを例に整理します。

## 参考資料

- [Using shadow DOM: Declaratively with HTML - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM#declaratively_with_html)
- [`<template>` - HTML Living Standard](https://html.spec.whatwg.org/multipage/scripting.html#the-template-element)
