---
title: "Shadow DOMのCSSは意図した拡張点だけを外へ開く"
---

Shadow DOMはページと内部のCSSセレクターを分けます。専用のセレクターを使えば、境界を保ったまま3つの対象をスタイルできます。ホスト自身、利用者がSlotに渡した内容、明示的に公開した内部部品です。

境界があっても、見た目を完全に固定する必要はありません。部品作者は既定の見た目を保ちながら、利用者が変更してよい範囲だけを明示できます。判断すべきなのは、どの接点を長期的なスタイルAPIとして維持するかです。

ホスト全体には`:host`、Slotへ渡された内容には`::slotted()`、内部の一部を明示的に公開する場合は`part`と`::part()`を使います。それぞれ選べる対象と責任が違います。

## :hostでCustom Element自身をスタイルする

Shadow Root内のCSSからホストを選ぶには`:host`を使います。

```css
:host {
  display: block;
  border: 1px solid #d7dce5;
  border-radius: 0.75rem;
  background: white;
}
```

Custom Elementの既定値は`display: inline`です。幅や高さを持つ部品では、意図するレイアウトを`:host`で明示します。

`hidden`属性も忘れないでください。`:host { display: block }`はブラウザの`[hidden] { display: none }`より優先される場合があります。

```css
:host([hidden]) {
  display: none;
}
```

## :host()で公開属性に応じた見た目を変える

ホストの属性は`:host()`で選べます。

```css
:host([completed]) .label {
  color: #667085;
  text-decoration: line-through;
}

:host([priority='high']) {
  border-inline-start: 0.3rem solid #dc2626;
}
```

状態をクラス名としてホストへ書き戻すと、利用者が設定した`class`を壊す恐れがあります。公開状態は属性または第23章のCustom Statesで表します。

## ページ側のテーマに反応するときは依存を狭くする

`:host-context()`を使うと、ホスト自身または祖先に一致する条件で内部スタイルを変えられます。

```css
:host-context([data-theme='dark']) {
  color: #f8fafc;
  background: #172033;
}
```

便利ですが、コンポーネントがページのDOM構造やクラス名へ依存します。テーマを渡す用途なら、CSS Custom Propertiesや`color-scheme`のほうが契約を明示しやすくなります。`:host-context()`は既存ページへ組み込むための限定的な橋として扱います。

## ::slotted()は直接割り当てられた要素だけを選ぶ

利用者がSlotへ渡した要素は`::slotted()`でスタイルできます。

```css
::slotted([slot='label']) {
  min-width: 0;
  overflow-wrap: anywhere;
}

::slotted(time) {
  color: #667085;
  font-size: 0.875rem;
}
```

`::slotted()`が選べるのは、Slotへ直接割り当てられた要素です。次の`strong`は選べません。

```html
<span slot="label"><strong>重要</strong>なタスク</span>
```

```css
/* strongは直接slottedではないため一致しない */
::slotted(span strong) {
  color: red;
}
```

利用者が渡した内容の内部までコンポーネントが管理しないことは、責務の境界としても妥当です。必要なら「slotへ渡すルート要素」の契約だけを文書化します。

## partで内部の選択した要素を公開する

利用者が内部要素を直接スタイルする必要がある場合、`part`属性で名前を付けます。

```html
<article part="container">
  <label part="control">
    <input type="checkbox" part="checkbox">
    <slot name="label"></slot>
  </label>
</article>
```

ページ側は`::part()`で選択します。

```css
task-item::part(container) {
  box-shadow: 0 0.25rem 1rem rgb(15 23 42 / 0.12);
}

task-item::part(checkbox) {
  inline-size: 1.25rem;
  block-size: 1.25rem;
}
```

内部のクラス名や要素階層を公開していない点が重要です。実装を`article`から`div`へ変更しても、`part="container"`を残せば利用側のCSSは保てます。

## 入れ子の部品ではexportpartsで名前を転送する

`<task-item>`内部で`<task-checkbox>`を使う場合、外側のCSSから内側のpartを直接たどれません。必要なpartだけを`exportparts`で転送します。

```html
<task-checkbox exportparts="control: checkbox"></task-checkbox>
```

利用側は外側の名前でスタイルできます。

```css
task-item::part(checkbox) {
  accent-color: #2563eb;
}
```

転送名を変えれば、内部コンポーネントの命名も隠せます。

## Custom PropertiesとPartsは目的が違う

色や余白の値を渡すならCSS Custom Properties、利用者が複数のCSS宣言や疑似クラスを自由に指定するならCSS Partsが向いています。

```css
task-item {
  --task-accent: #7c3aed;
}

task-item::part(container):hover {
  transform: translateY(-1px);
}
```

Partsを増やしすぎると、内部実装を変更する自由が減ります。「その部分を利用者が独立してスタイルする必要があるか」を問い、契約として維持できる名前だけを公開します。

次章では、Custom Propertiesをデザイントークンとして整理し、同じStylesheetを複数のShadow Rootで共有します。

## 参考資料

- [CSS scoping - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Scoping)
- [CSS Shadow Parts Module](https://www.w3.org/TR/css-shadow-parts-1/)
