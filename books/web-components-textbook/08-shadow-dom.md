---
title: "Shadow DOMは内部構造を変更できる境界を作る"
---

Shadow DOMを使う最大の利点は、利用者が内部のタグ構造へ依存しにくくなることです。ページのCSSが内部要素へ偶然当たる事故を減らし、コンポーネント側のCSSが外へ漏れることも防ぎます。

ここで作るのは「別の文書」ではなく、1つのDOM要素に結び付いた内部ツリーです。ホストは通常のDocument Treeに残り、その内側にShadow Rootを入口とするShadow Treeが加わります。画面上では1つの部品に見えても、CSSセレクターとDOM探索には境界があります。

Shadow DOMはprivate fieldのような完全な秘匿機構ではありません。`open`なら外部JavaScriptから参照でき、イベントやSlotも境界を越えて関係します。役割は内部を秘密にすることではなく、「どこまでを変更可能な実装詳細とするか」をブラウザにも理解できる形で示すことです。

また、すべてのCustom ElementにShadow DOMが必要なわけではありません。ページ側から子要素を直接編集させたい部品や、既存CSSとの統合を優先する部品ではLight DOMだけを使う選択もあります。

## ホストへShadow Rootを取り付ける

Shadow DOMを持つ通常の要素をShadow Hostと呼びます。Custom Element自身をホストにする形が一般的です。

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

この性質を使うと、ページ全体の文字設定に馴染ませつつ、部品内部のレイアウトは守れます。テーマ設計は第12章で扱います。

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

この方法はサーバーレンダリングと初期表示に関わるため、第22章で扱います。クライアント側の基本実装では`attachShadow()`を使います。

次章では、利用者が書いたLight DOMをShadow Treeへ表示するSlotと、内部構造を再利用するTemplateを学びます。

## 参考資料

- [Using shadow DOM - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM)
- [Shadow trees - DOM Standard](https://dom.spec.whatwg.org/#shadow-trees)
