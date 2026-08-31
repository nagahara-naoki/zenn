---
title: "ElementInternalsとフォーム連携"
---

Form-associated Custom Elementは、`input`や`select`と同じようにHTMLフォームの一員として扱われるCustom Elementです。送信データへの値の提供、制約検証、`form.reset()`への応答、`fieldset`による無効化まで、ブラウザが組み込み入力へ用意している仕組みに参加できます。クラスへ`static formAssociated = true`を宣言すると、その要素はフォームの一員になります。

`ElementInternals`は、要素が自分の内部状態をブラウザへ申告するための窓口です。`attachInternals()`が返すオブジェクトで、送信値、妥当性、関連するフォームやラベル、そしてアクセシビリティ上の役割や状態を扱います。Shadow DOMの外から触れない内部情報を、要素自身の責任でブラウザへ渡す口だと考えると位置づけがはっきりします。

この2つを組み合わせる理由は、フォームが値を読むだけの仕組みではないからです。どの要素がどの`form`に所属するか、送信データへ何を含めるか、`required`などの制約を満たすか、`fieldset`が無効になったか。これらをブラウザがまとめて管理しています。内部に`input`を置くだけでは、外側のフォームから見える要素は`<task-input>`のままです。`ElementInternals`を通して値と妥当性を渡すと、`FormData`、`form.reset()`、制約検証と同じ流れへ入れます。

この章では、タスク名を入力する`<task-input>`を題材に、フォーム参加の宣言、送信値の受け渡し、制約検証、フォームのライフサイクルへの応答を順に実装します。

## 組み込みinputで足りるかを先に確かめる

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

## まとめ

この章では、独自入力をHTMLフォームへ参加させる方法を学びました。

- Form-associated Custom Elementは、`static formAssociated = true`を宣言したCustom Elementです。組み込み入力と同じフォームの仕組みに参加できます。
- `ElementInternals`は`attachInternals()`で取得し、送信値や妥当性をブラウザへ申告する窓口になります。
- `setFormValue()`が`FormData`へ載る値を決めます。文字列、`File`、`FormData`、`null`を渡せます。
- `setValidity()`でValidityStateとエラーメッセージ、フォーカス先を設定すると、制約検証の流れに入れます。
- `formDisabledCallback()`、`formResetCallback()`、`formStateRestoreCallback()`で、無効化、リセット、状態復元へ応答します。
- `internals.form`と`internals.labels`を公開すると、組み込み要素に近い使い勝手になります。
- ブラウザの検証UIだけに頼らず、エラー文は画面にも表示します。

ここまでで、Custom Elementの登録からデータの出入り、Shadow DOM、スタイル、アクセシビリティ、フォームまでがつながりました。まずUIを完成させたいなら『タスク管理UIを完成させる』の章へ進んでください。配布や運用まで学ぶなら、次章の『セキュリティとXSS対策』から『Litとは何か』の章まで読み進めます。

## 参考資料

- [Form-associated custom elements - HTML Living Standard](https://html.spec.whatwg.org/multipage/custom-elements.html#custom-elements-face-example)
- [ElementInternals.setFormValue() - MDN](https://developer.mozilla.org/en-US/docs/Web/API/ElementInternals/setFormValue)
