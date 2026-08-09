---
title: "ネイティブ要素を再利用して操作と意味を守る"
---

Shadow DOMで見た目を閉じても、アクセシビリティは閉じてはいけません。キーボード操作、フォーカス表示、アクセシブルな名前、状態の通知は、ページ全体の利用体験としてつながる必要があります。

独自のタグ名には、組み込み要素のような意味が最初からありません。`<task-item>`と書いただけでは、それがチェック可能なのか、ボタンなのか、一覧項目なのかを支援技術は判断できません。内部で適切なネイティブ要素を使うか、ホストへ意味と状態を与える必要があります。

見た目を再現することと、操作体系を再現することは別です。`div`をボタンらしく装飾しても、EnterやSpaceでの操作、無効状態、フォーカス移動、アクセシブルな名前は自動では付きません。本章では、まずブラウザが持つ意味を再利用し、足りない部分だけを補います。

## divへ役割を足す前にbuttonを使う

クリックできる見た目を`div`で作ると、キーボード操作と支援技術向けの意味を自分で実装する必要があります。

```html
<div class="remove">削除</div>
```

Shadow DOM内部で通常の`button`を使えば、Enter、Space、フォーカス、無効状態、roleをブラウザが提供します。

```html
<button type="button" part="remove-button">
  削除
</button>
```

Custom Elementは新しい見た目を作る道具ですが、既存要素の振る舞いまで捨てる理由にはなりません。

## アクセシブルな名前を画面上のラベルから作る

`<task-input>`では、内部の`label`にSlotと`input`を入れます。

```html
<label>
  <span class="label"><slot name="label">タスク名</slot></span>
  <input type="text" part="input">
</label>
```

```html
<task-input>
  <span slot="label">新しいタスク</span>
</task-input>
```

画面に表示されるラベルがそのまま入力欄の名前になります。`aria-label`だけに頼ると、見える説明と支援技術が読む名前がずれることがあります。

アイコンだけの削除ボタンには、目に見えない名前を付けます。

```html
<button type="button" aria-label="タスクを削除">
  <svg aria-hidden="true" viewBox="0 0 24 24"><!-- ... --></svg>
</button>
```

## フォーカス順は文書の順番に任せる

通常の操作要素は`tabindex="0"`で文書順に並びます。`tabindex="1"`以上を使うと、DOMの順番とフォーカス順が離れ、画面を見ている人にも予測しにくくなります。

カスタム要素自身をフォーカス可能にするより、内部の`button`や`input`へフォーカスさせるほうが、操作対象を明確にできます。

外部から`taskInput.focus()`を呼んだとき内部入力へ移したい場合、ホストの`focus()`を実装します。

```ts
class TaskInput extends HTMLElement {
  #input: HTMLInputElement;

  focus(options?: FocusOptions): void {
    this.#input.focus(options);
  }
}
```

組み込みの`HTMLElement.focus()`と同じ名前と引数を保つため、利用者は通常の要素と同じ感覚で扱えます。

## delegatesFocusを使う前にフォーカス対象を確認する

Shadow Root作成時の`delegatesFocus`を`true`にすると、ホストの非フォーカス領域をクリックした場合やホストへフォーカスを移した場合に、内部のフォーカス可能な要素へ委譲できます。

```ts
const root = this.attachShadow({
  mode: 'open',
  delegatesFocus: true,
});
```

最初のフォーカス可能要素が常に主操作とは限りません。削除ボタンが先にある構造で委譲すると、予期しない場所へ移る可能性があります。内部のDOM順とフォーカス順を確認してから使います。

## :focus-visibleでフォーカスを消さない

キーボード利用者が現在地を把握できるように、フォーカス表示を残します。

```css
button:focus-visible,
input:focus-visible {
  outline: 0.2rem solid var(--task-focus-color, #2563eb);
  outline-offset: 0.15rem;
}
```

`outline: none`だけを書くと現在地が消えます。ブランドに合わせて変更するなら、同等以上に見える代替表示を用意します。

## 複合部品ではroving tabindexを使う

3つのフィルターボタンをTabキーですべて回ると、ページの移動回数が増えます。ひとまとまりの選択肢なら、Tabでグループへ入り、矢印キーで内部を移動する設計が使えます。これをroving tabindexと呼びます。

```ts
#moveFocus(offset: number): void {
  const buttons = [...this.#root.querySelectorAll('button')];
  const current = buttons.indexOf(
    this.#root.activeElement as HTMLButtonElement,
  );
  const next = (current + offset + buttons.length) % buttons.length;

  buttons.forEach((button, index) => {
    button.tabIndex = index === next ? 0 : -1;
  });

  buttons[next]?.focus();
}
```

`keydown`でArrowLeftとArrowRightを受け、選択状態と`aria-pressed`を同期します。キー操作はWAI-ARIA Authoring Practicesの該当パターンと照らし、独自の組み合わせを増やしません。

## 状態は色以外でも伝える

完了済みタスクを灰色にするだけでは、色を区別しにくい利用者へ伝わりません。チェックボックスの`checked`、取り消し線、テキストなど複数の手がかりを組み合わせます。

読み込み中の通知には`aria-live`を使えますが、頻繁な更新をすべて読み上げると操作を妨げます。失敗や保存完了など、利用者が次の判断に必要な変化へ絞ります。

## ElementInternalsはホストの既定セマンティクスを持てる

内部に適切なネイティブ要素を置けないCustom Elementでは、`attachInternals()`から得た`ElementInternals`へ既定のroleやARIA状態を設定できます。

```ts
class ToggleChip extends HTMLElement {
  #internals = this.attachInternals();

  constructor() {
    super();
    this.#internals.role = 'button';
    this.#internals.ariaPressed = 'false';
  }
}
```

利用者がホストへ明示したARIA属性は、コンポーネントの既定値より優先できます。ただしroleを設定しただけでキーボード操作が生まれるわけではありません。可能ならネイティブ`button`を内部で使い、ElementInternalsはフォーム参加やCustom Element自身の意味が必要な場面へ絞ります。

アクセシビリティはコードレビューだけで完了しません。キーボードだけで操作し、ブラウザのアクセシビリティツリーを確認し、可能なら実際のスクリーンリーダーでも試します。テストの自動化は第19章で扱います。

次章では、`ElementInternals`を使ってCustom ElementをHTMLフォームへ参加させます。

## 参考資料

- [ElementInternals - MDN](https://developer.mozilla.org/en-US/docs/Web/API/ElementInternals)
- [WAI-ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/)
