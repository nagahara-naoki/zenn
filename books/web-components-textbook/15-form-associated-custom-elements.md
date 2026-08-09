---
title: "ElementInternalsで独自入力をHTMLフォームへ参加させる"
---

通常のCustom Elementへ`name`と`value`を付けても、`FormData`には含まれません。HTMLフォームの送信、制約検証、リセット、無効化の仕組みに参加するには、Form-associated Custom Elementとして定義します。

HTMLフォームは、入力欄を見つけて値を読むだけの仕組みではありません。どの要素がどの`form`に所属するか、送信データへ何を含めるか、`required`などの制約を満たすか、`fieldset`が無効になったかをブラウザがまとめて管理します。

Form-associated Custom Elementは、この管理に独自要素を参加させます。内部に`input`を置くだけでは、外側のフォームから見える要素は`<task-input>`のままです。`ElementInternals`を通して値と妥当性をブラウザへ渡すことで、`FormData`、`form.reset()`、制約検証と同じ流れに入れます。

## まず組み込みinputで足りないかを確かめる

フォーム部品の再実装には、入力方式、キーボード、モバイル、オートフィル、検証、支援技術への対応が伴います。見た目を変えたいだけなら、通常の`input`とCSSを使うほうが安全です。

Form-associated Custom Elementが役立つのは、複数の内部入力を1つの値として送る部品や、製品をまたいで同じ入力契約を配布する場合です。本章では仕組みを学ぶため、内部の`input`を包む`<task-input>`を作ります。

## static formAssociatedでフォーム参加を宣言する

```ts:src/task-input.ts
const template = document.createElement('template');
template.innerHTML = `
  <style>
    :host { display: block; }
    :host([hidden]) { display: none; }
    label { display: grid; gap: 0.35rem; }
  </style>
  <label>
    <span><slot name="label">タスク名</slot></span>
    <input type="text" part="input">
  </label>
`;

export class TaskInput extends HTMLElement {
  static formAssociated = true;
  static observedAttributes = ['required'];

  #internals: ElementInternals;
  #input: HTMLInputElement;
  #defaultValue = '';
  #initialized = false;

  constructor() {
    super();

    this.#internals = this.attachInternals();
    const root = this.attachShadow({ mode: 'open' });
    root.append(template.content.cloneNode(true));

    const input = root.querySelector('input');
    if (!input) throw new Error('inputが見つかりません');
    this.#input = input;
  }
}
```

`static formAssociated = true`を宣言したクラスだけが、フォーム用の`ElementInternals` APIを利用できます。

## setFormValueで送信値をブラウザへ渡す

`value`プロパティを用意し、内部入力とフォーム値を同期します。

```ts
get value(): string {
  return this.#input.value;
}

set value(value: string) {
  this.#input.value = value;
  this.#internals.setFormValue(value);
  this.#validate();
}

connectedCallback(): void {
  if (!this.#initialized) {
    this.#defaultValue = this.getAttribute('value') ?? '';
    this.value = this.#defaultValue;
    this.#initialized = true;
  }

  this.#input.addEventListener('input', this.#handleInput);
}

disconnectedCallback(): void {
  this.#input.removeEventListener('input', this.#handleInput);
}

#handleInput = (): void => {
  this.#internals.setFormValue(this.#input.value);
  this.#validate();

  this.dispatchEvent(
    new Event('input', { bubbles: true, composed: true }),
  );
};
```

`setFormValue()`の第1引数には、文字列、`File`、`FormData`、`null`を渡せます。`null`ならフォーム送信から除外されます。複数の値を送る部品では`FormData`が使えます。

利用方法は組み込み入力とほぼ同じです。

```html
<form id="task-form">
  <task-input name="label" required>
    <span slot="label">新しいタスク</span>
  </task-input>
  <button type="submit">追加</button>
</form>
```

```ts
const form = document.querySelector<HTMLFormElement>('#task-form');

form?.addEventListener('submit', (event) => {
  event.preventDefault();
  const data = new FormData(form);
  console.log(data.get('label'));
});
```

## setValidityで制約検証へ参加する

`required`属性があり、入力が空ならフォームを無効にします。

```ts
get required(): boolean {
  return this.hasAttribute('required');
}

set required(value: boolean) {
  this.toggleAttribute('required', value);
}

attributeChangedCallback(name: string): void {
  if (name !== 'required') return;
  this.#input.required = this.required;
  this.#validate();
}

#validate(): void {
  const missing = this.required && this.value.trim() === '';

  if (missing) {
    this.#internals.setValidity(
      { valueMissing: true },
      'タスク名を入力してください',
      this.#input,
    );
    return;
  }

  this.#internals.setValidity({});
}
```

第3引数の内部`input`は、検証エラー時のフォーカス先となる要素です。エラーが解消したら空のオブジェクトでValidityStateをクリアします。

Custom Element側にも、組み込み入力と似た検証APIを公開できます。

```ts
checkValidity(): boolean {
  return this.#internals.checkValidity();
}

reportValidity(): boolean {
  return this.#internals.reportValidity();
}

focus(options?: FocusOptions): void {
  this.#input.focus(options);
}
```

## フォームのライフサイクルへ応答する

Form-associated Custom Elementには、フォームとの関係に応じたコールバックがあります。

```ts
formDisabledCallback(disabled: boolean): void {
  this.#input.disabled = disabled;
}

formResetCallback(): void {
  this.value = this.#defaultValue;
}

formStateRestoreCallback(
  state: string | File | FormData | null,
): void {
  if (typeof state === 'string') {
    this.value = state;
  }
}
```

`formDisabledCallback()`は、自身の`disabled`属性だけでなく、無効な`fieldset`内に入った場合にも呼ばれます。`formResetCallback()`は`form.reset()`に応答します。

ブラウザがナビゲーション後やオートフィルで状態を復元するときは`formStateRestoreCallback()`が使われます。送信値と復元用状態を分けたい場合、`setFormValue(value, state)`の第2引数へ状態を渡します。

## formとlabelsを公開する

`ElementInternals`から関連フォームとラベルを取得できます。

```ts
get form(): HTMLFormElement | null {
  return this.#internals.form;
}

get labels(): NodeList {
  return this.#internals.labels;
}
```

組み込み要素に近いプロパティ名をそろえると、フォームライブラリや利用者が予測しやすくなります。

## エラー文は見える場所にも表示する

ブラウザの検証UIだけに頼ると、表示方法や読み上げが環境で変わります。コンポーネント内にエラー表示領域を置き、`aria-describedby`または関連要素APIで入力と結びます。

```html
<p id="error" part="error" aria-live="polite"></p>
```

送信を止める規則、視覚的な説明、支援技術への通知を同じ検証結果から更新すると、状態がずれません。

これで通読部は完了です。Custom Elementの登録から始まり、データの出入り、Shadow DOM、スタイル、アクセシビリティ、フォームまでつながりました。まずUIを完成させるなら第25章へ進んでください。配布・運用まで学ぶなら、次章から第24章までを続けます。

## 参考資料

- [Form-associated custom elements - HTML Living Standard](https://html.spec.whatwg.org/multipage/custom-elements.html#custom-elements-face-example)
- [ElementInternals.setFormValue() - MDN](https://developer.mozilla.org/en-US/docs/Web/API/ElementInternals/setFormValue)
