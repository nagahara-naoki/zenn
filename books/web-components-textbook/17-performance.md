---
title: "性能は要素数と更新頻度を測ってから改善する"
---

Web Componentsだから速い、Shadow DOMだから遅いという一言では性能を判断できません。負荷を決めるのは、作るDOMノードの数、更新回数、レイアウトを読む回数、保持したリスナーやObserverの数です。

## まず対象の操作を計測する

`performance.mark()`と`performance.measure()`を使うと、要素生成や更新にかかった時間を開発者ツールへ記録できます。

```ts
performance.mark('task-list:start');

list.tasks = createTasks(1_000);
await list.updateComplete;

performance.mark('task-list:end');
performance.measure(
  'task-list render',
  'task-list:start',
  'task-list:end',
);
```

ネイティブ実装に`updateComplete`がなければ、描画完了を返すPromiseを公開するか、テスト用に次のanimation frameを待ちます。「体感で遅い」という感覚を、再現可能な操作と数値に置き換えることが大切です。

Chrome、Firefox、SafariのPerformanceツールでは、Script、Style、Layout、Paintのどこに時間を使ったか確認できます。

## 内部DOMを安定させる

第7章で実装したように、内部要素は一度作り、値だけを更新します。

```ts
#render(): void {
  this.#checkbox.checked = this.#task?.completed ?? false;
  this.#label.textContent = this.#task?.label ?? '';
}
```

毎回`innerHTML`で置き換えると、ノード生成、解析、リスナー登録、フォーカス復元が増えます。変更箇所が少ない部品では、参照を保持してDOMプロパティだけ更新する方法が単純で高速です。

一覧全体の差分計算が複雑になったら、手作業の更新器を膨らませず、Litなどのテンプレートライブラリを検討します。

## 同じ処理内の更新をまとめる

プロパティsetterのたびに同期描画すると、3つの値を設定しただけで3回更新されます。`queueMicrotask()`で1回へまとめます。

```ts
#requestUpdate(): void {
  if (this.#updatePending) return;
  this.#updatePending = true;

  queueMicrotask(() => {
    this.#updatePending = false;
    this.#render();
  });
}
```

DOMを書き換えた直後に`getBoundingClientRect()`を読み、また書き換える処理を繰り返すと、同期Layoutが発生しやすくなります。読み取りをまとめ、その後で書き込みをまとめます。

## Stylesheetを共有する

同じCSSを持つ要素が大量にある場合、Constructable Stylesheetを共有できます。

```ts
const sheet = new CSSStyleSheet();
sheet.replaceSync(styles);

class TaskItem extends HTMLElement {
  constructor() {
    super();
    const root = this.attachShadow({ mode: 'open' });
    root.adoptedStyleSheets = [sheet];
  }
}
```

CSS文字列を各インスタンスで作り直さないことも大切です。`template`や`CSSStyleSheet`はモジュールスコープで一度作り、インスタンスから再利用します。

## イベントは祖先で受け取る

1,000件の`<task-item>`それぞれへ一覧側のリスナーを付ける代わりに、bubbleする`task-toggle`を`<task-list>`で受け取れます。

```ts
list.addEventListener('task-toggle', handleTaskToggle);
```

ただし、内部チェックボックス自身のリスナーまで無理に外へ集約する必要はありません。リスナー数だけで性能問題になることは少なく、所有関係が明確なほうが保守しやすい場合もあります。計測結果と実装の複雑さを合わせて判断します。

## 非表示の機能は必要になるまで読み込まない

ページ下部の重いチャートやエディタは、表示が近づいてから定義モジュールを読み込めます。

```ts
const target = document.querySelector('task-analytics');

const observer = new IntersectionObserver(async ([entry]) => {
  if (!entry?.isIntersecting) return;

  await import('./task-analytics.js');
  observer.disconnect();
});

if (target) observer.observe(target);
```

定義前にも読めるLight DOMのフォールバックを入れておくと、JavaScript待ちで空白になりません。

```html
<task-analytics>
  <p>分析データを読み込んでいます。</p>
</task-analytics>
```

`:defined`と`:not(:defined)`で、アップグレード前後の見た目を調整できます。

```css
task-analytics:not(:defined) {
  display: block;
  min-block-size: 12rem;
}
```

コンテンツ自体を`display: none`で隠すと、読み込み失敗時に何も残りません。レイアウトの揺れを抑えつつ、フォールバックは見える状態にします。

## 切断時に外部参照を解放する

性能劣化が時間とともに大きくなる場合、描画速度よりメモリーリークを疑います。

- `window`や`document`へ登録したイベント
- 終了していないタイマー
- `ResizeObserver`や`MutationObserver`
- モジュールスコープの配列へ保存した要素参照

第5章の`AbortController`と`disconnect()`を使い、要素が外れた後に処理を続けないようにします。Memoryパネルで要素を削除した後も参照が残っていないか確認します。

## 大量一覧の責務を1件の要素へ押し込まない

数万件のタスクを扱う場合、1件を描画する`<task-item>`だけを最適化しても限界があります。画面に見える範囲だけ描画するvirtualizationは、一覧を所有する`<task-list>`またはアプリケーションの責務です。

部品単体の改善と、画面全体のデータ量を減らす改善を分けて測ります。最も効く施策は、作らなくてよいDOMを作らないことです。

## 性能レビューの順番

実務では次の順で調べると、手当たり次第の最適化を避けられます。

1. 遅い操作と対象端末を固定する
2. Performance記録でScript、Layout、Paintの比率を見る
3. 要素数と更新回数を数える
4. DOMの全置換、同期Layout、外部参照を減らす
5. 同じ操作を再計測する

性能は実装方式の評判ではなく、対象操作の計測結果で判断します。
