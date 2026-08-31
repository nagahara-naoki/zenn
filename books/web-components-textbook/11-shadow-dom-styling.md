---
title: "Shadow DOMのスタイリング"
---

Shadow DOMは、CSSの届く範囲をページと内部で分けます。まず、どのCSSが何に当たり、何に当たらないかを整理します。

- ページ側のCSSは、Shadow Root内部の要素に当たりません。`.container`という同じクラス名がページにあっても、内部の`.container`は影響を受けません。
- Shadow Root内部のCSSは、外のページに当たりません。内部で`p { color: red }`と書いても、ページ本体の`p`は変わりません。
- 継承するプロパティは境界を越えます。`color`や`font-family`、CSS Custom Propertiesの値は、ホストから内部へ受け継がれます。
- Slotへ渡された内容は、利用者側のDOMに属します。表示位置は内部でも、スタイルの主導権はページ側にあります。

この分離があるおかげで、部品の見た目は外から壊されません。ただし完全に固定してしまうと、利用者は何も調整できなくなります。そこで、部品作者が「ここは変えてよい」と決めた接点だけを外へ開くための専用セレクターが用意されています。

| セレクター | 選べる対象 | 書く場所 |
|---|---|---|
| `:host`、`:host()` | ホスト要素自身 | Shadow Root内部 |
| `::slotted()` | Slotへ直接渡された要素 | Shadow Root内部 |
| `::part()` | `part`属性で公開された内部要素 | ページ側 |

順に見ていきます。

## :hostでホスト自身を選ぶ

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

## :host()で属性ごとに見た目を変える

ホストの属性を条件にするには、括弧付きの`:host()`を使います。

```css
:host([completed]) .label {
  color: #667085;
  text-decoration: line-through;
}

:host([priority='high']) {
  border-inline-start: 0.3rem solid #dc2626;
}
```

状態をクラス名としてホストへ書き戻すと、利用者が設定した`class`を壊す恐れがあります。公開する状態は属性か、『Custom StatesとScoped Registries』の章で扱うCustom Statesで表します。

## :host-context()は限定的に使う

`:host-context()`を使うと、ホスト自身または祖先に一致する条件で内部スタイルを変えられます。

```css
:host-context([data-theme='dark']) {
  color: #f8fafc;
  background: #172033;
}
```

便利な反面、コンポーネントがページ側のDOM構造やクラス名へ依存します。テーマを渡したいだけなら、CSS Custom Propertiesや`color-scheme`のほうが契約を明示できます。`:host-context()`は、既存ページへ組み込むための限定的な橋として扱います。ブラウザによっては未対応な点にも注意が必要です。

## ::slotted()でSlotの内容を選ぶ

利用者がSlotへ渡した要素は、`::slotted()`でスタイルできます。

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

選べるのは、Slotへ直接割り当てられた要素だけです。次の`strong`は`span`の子なので選べません。

```html
<span slot="label"><strong>重要</strong>なタスク</span>
```

```css
/* strongは直接slottedではないため一致しない */
::slotted(span strong) {
  color: red;
}
```

利用者が渡した内容の内部までコンポーネントが管理しないのは、責務の境界として妥当です。必要なら「slotへ渡すルート要素」の契約だけを文書化します。なお、`::slotted()`で指定したスタイルはページ側の指定に負けます。あくまで既定値を与えるための手段です。

## partで内部要素を公開する

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

公開しているのは名前だけで、内部のクラス名や要素階層ではありません。実装を`article`から`div`へ変更しても、`part="container"`さえ残せば利用側のCSSは動き続けます。

## exportpartsで入れ子のpartを転送する

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

## Custom PropertiesとPartsの使い分け

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

## まとめ

この章では、Shadow DOMのスタイリングを学びました。

- ページ側のCSSは内部へ届かず、内部のCSSも外へ出ません。ただし`color`などの継承プロパティは境界を越えます。
- `:host`はホスト自身を選びます。`display`と`[hidden]`の指定を忘れないようにします。
- `:host()`はホストの属性に応じた分岐に使います。`:host-context()`はページ構造への依存を招くため限定的に使います。
- `::slotted()`が選べるのは、Slotへ直接割り当てられた要素だけです。
- `part`と`::part()`を使うと、内部構造を隠したまま特定の要素をスタイル可能にできます。入れ子の部品は`exportparts`で転送します。
- 値を渡すならCustom Properties、自由なスタイル指定を許すならPartsを選びます。

次章では、Custom Propertiesをテーマとして整理し、同じStylesheetを複数のShadow Rootで共有する方法を学びます。

## 参考資料

- [CSS scoping - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Scoping)
- [CSS Shadow Parts Module](https://www.w3.org/TR/css-shadow-parts-1/)
