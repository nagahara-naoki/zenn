---
title: "Shadow DOMとは何か"
---

Shadow DOMとは、1つのDOM要素の内側に、外部のCSSとDOM探索から隔離された内部ツリーを作る仕組みです。作られる内部ツリーをShadow Tree、その入口となるノードをShadow Root、内部ツリーを抱える側の要素をShadow Hostと呼びます。

ホストは通常のDocument Treeに残り、その内側にShadow Rootを起点とするShadow Treeがぶら下がります。画面上では1つの部品に見えますが、ページ側のCSSセレクターとDOM探索が届くのはホストまでで、その内側のShadow Treeには届きません。

用語の関係を整理します。

| 用語 | 指すもの |
|---|---|
| Shadow Host | 内部ツリーを抱える側の要素。`<task-item>`のような通常のDOM要素 |
| Shadow Root | 内部ツリーの入口となるノード。`attachShadow()`の戻り値 |
| Shadow Tree | Shadow Rootの下にある、作者が管理する内部構造 |
| Light DOM | ホストの子として利用者がHTMLに書いた要素 |

この境界があると、利用者は内部のタグ構造へ依存しにくくなります。ページのCSSが内部要素へ偶然当たる事故が減り、コンポーネント側のCSSが外へ漏れることも防げます。内部の作りを後から変更しても、使う側のコードが壊れにくくなるわけです。

ただし、Shadow DOMはprivate fieldのような完全な秘匿機構ではありません。`open`なら外部JavaScriptから参照でき、イベントやSlotも境界を越えて関係します。担っているのは内部の秘匿ではなく、「どこまでを変更可能な実装詳細とするか」をブラウザにも理解できる形で示す役割です。

## Shadow Rootを取り付ける

Shadow Rootは`attachShadow()`で取り付けます。Custom Element自身をホストにする形が一般的です。

```ts
class TaskItem extends HTMLElement {
  constructor() {
    super();

    const root = this.attachShadow({ mode: 'open' });
    const label = document.createElement('label');
    const checkbox = document.createElement('input');
    const text = document.createElement('span');

    checkbox.type = 'checkbox';
    text.textContent = '未設定';
    label.append(checkbox, text);
    root.append(label);
  }
}
```

`attachShadow()`が返す`ShadowRoot`は、内部ツリーの入口です。`DocumentFragment`に似ていますが、ホストとの関係、Slot、スタイル境界を持ちます。

## openは外部から入口を取得できる

`mode: 'open'`で作ると、ホストの`shadowRoot`プロパティから内部へアクセスできます。

```ts
const item = document.querySelector('task-item');
const checkbox = item?.shadowRoot?.querySelector('input');
```

`mode: 'closed'`では`item.shadowRoot`が`null`になります。ただし、closedはセキュリティ境界ではありません。コンストラクターが返り値を保存すれば内部コードからアクセスでき、ブラウザ拡張や開発者ツールまで完全に遮断する仕組みでもありません。

通常はopenを選ぶと、テストやデバッグ、ほかの標準APIとの連携が簡単です。closedを選ぶなら、外部アクセスを拒むことで得られる具体的な利点を先に決めます。

## CSSセレクターは境界を越えない

ページ側に次のCSSがあっても、Shadow Tree内の`input`には一致しません。

```css
input {
  appearance: none;
  border: 10px solid red;
}
```

Shadow Root内部のCSSもページへ漏れません。

```ts
const style = document.createElement('style');
style.textContent = `
  label {
    display: flex;
    gap: 0.5rem;
    align-items: center;
  }
`;

root.append(style, label);
```

境界があるおかげで、`.label`のような短いクラス名でもページ側と衝突しません。

## 継承するCSSプロパティは境界を越える

Shadow DOMはCSSを完全に隔離するわけではありません。`color`や`font-family`のような継承プロパティは、ホストから内部へ引き継がれます。CSS Custom Propertiesも継承します。

```css
task-item {
  color: #172033;
  font-family: system-ui, sans-serif;
  --task-accent: #2563eb;
}
```

```css
input {
  accent-color: var(--task-accent);
}
```

この性質を使うと、ページ全体の文字設定に馴染ませつつ、部品内部のレイアウトは守れます。テーマ設計は『テーマとスタイルシートの共有』の章で扱います。

## JavaScriptの探索範囲も分かれる

文書側の`document.querySelector()`は、Shadow Tree内部の要素を見つけません。

```ts
document.querySelector('task-item input'); // null
```

内部を探すときは、対象の`ShadowRoot`から始めます。

```ts
item.shadowRoot?.querySelector('input');
```

この境界により、ページ側の処理がコンポーネント内部のクラス名へ偶然依存する事故を減らせます。外部へ操作を許す必要があるなら、内部要素を直接触らせず、ホストのメソッドやプロパティとして公開します。

## Shadow DOMを使わない選択もある

すべてのCustom ElementにShadow DOMが必要なわけではありません。次の要素はLight DOMだけで作る判断もあり得ます。

- ページ側のCSSで中身を自由にレイアウトしたい薄いラッパー
- 子要素の意味構造を文書ツリーへそのまま残したい要素
- JavaScriptの振る舞いだけを追加する要素

Shadow DOMを使うと、内部を守る代わりにスタイルや参照の公開口を設計する仕事が増えます。内部実装を変更できる境界が必要かどうかで選びます。

## 宣言的なShadow DOMはSSRで使う

JavaScriptを実行せずHTMLだけでShadow Rootを作るDeclarative Shadow DOMも標準化されています。

```html
<task-item>
  <template shadowrootmode="open">
    <span>サーバーで生成した内容</span>
  </template>
</task-item>
```

この方法はサーバーレンダリングと初期表示に関わるため、『Declarative Shadow DOMとSSR』の章で扱います。クライアント側の基本実装では`attachShadow()`を使います。

## まとめ

この章では、Shadow DOMが作る境界を学びました。

- Shadow DOMは、1つのDOM要素の内側に、外部のCSSとDOM探索から隔離された内部ツリーを作る仕組みです。
- ホストがShadow Host、内部ツリーの入口がShadow Root、その下の構造がShadow Treeです。
- Shadow Rootは`attachShadow({ mode: 'open' })`で取り付けます。
- `open`ならホストの`shadowRoot`から内部へアクセスでき、`closed`でも秘匿にはなりません。
- CSSセレクターとDOM探索は境界を越えませんが、`color`などの継承プロパティとCSS Custom Propertiesは越えます。
- Light DOMだけで作るほうが素直な要素もあるため、内部を守る必要があるかで選びます。

次章では、利用者が書いたLight DOMをShadow Treeへ表示するSlotと、内部構造を再利用するTemplateを学びます。

## 参考資料

- [Using shadow DOM - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM)
- [Shadow trees - DOM Standard](https://dom.spec.whatwg.org/#shadow-trees)
