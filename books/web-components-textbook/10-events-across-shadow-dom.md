---
title: "内部の操作はCustomEventで外へ通知する"
---

コンポーネントの外から内へ値を渡す手段が属性とプロパティなら、内から外へ起きたことを伝える手段はイベントです。DOM標準の通知モデルに合わせると、Vanilla JavaScriptでもフレームワークでも同じ契約を利用できます。

コールバック関数をプロパティで渡す方法でも通知はできます。ただし、DOMにはすでに複数の祖先が同じ出来事を受け取れるイベント伝播の仕組みがあります。Custom Elementがイベントを使えば、利用者は組み込み要素と同じ`addEventListener()`で購読でき、一覧側で子要素の操作をまとめて扱えます。

イベントが表すのは、内部DOMで発生した低水準の操作ではなく、コンポーネントとして外へ伝える出来事です。内部ボタンの`click`をそのまま公開するのではなく、「タスクの完了状態を変えたい」という`task-toggle`へ翻訳します。この翻訳によって、内部をボタンから別のUIへ変えても公開契約を保てます。

## CustomEventのdetailへ必要な情報だけを入れる

`<task-item>`内部のチェックボックスが変わったら、`task-toggle`イベントをホストから通知します。

```ts
this.#checkbox.addEventListener('change', () => {
  const task = this.#task;
  if (!task) return;

  this.dispatchEvent(
    new CustomEvent('task-toggle', {
      bubbles: true,
      composed: true,
      detail: {
        id: task.id,
        completed: this.#checkbox.checked,
      },
    }),
  );
});
```

利用側は通常の`addEventListener()`で受け取ります。

```ts
document.addEventListener('task-toggle', (event) => {
  if (!(event instanceof CustomEvent)) return;

  console.log(event.detail.id);
  console.log(event.detail.completed);
});
```

`detail`へ内部要素の参照を入れると、利用側が実装へ依存します。公開するのは、状態更新に必要な識別子と値だけにします。

## bubblesとcomposedは別の境界を制御する

`bubbles`は親方向へ伝播するかを決めます。`composed`はShadow DOMの境界を越えられるかを決めます。

| `bubbles` | `composed` | 結果 |
|---|---|---|
| `false` | `false` | 発生元だけで処理される |
| `true` | `false` | 同じShadow Tree内を伝播する |
| `false` | `true` | 境界を越えられるが通常のbubbleはしない |
| `true` | `true` | Shadow DOMを越えて祖先へ伝播する |

アプリケーションへ公開する操作イベントには、通常`bubbles: true`と`composed: true`を指定します。内部だけで使うイベントは`composed: false`のままにし、実装詳細を境界内へ留めます。

## event.targetは境界でホストへ置き換わる

Shadow DOM内部の`input`から`change`イベントが届いたとき、外部から見える`event.target`は`<task-item>`に置き換わります。この処理をretargetingと呼びます。

```ts
document.addEventListener('click', (event) => {
  console.log(event.target); // <task-item>になる場合がある
  console.log(event.composedPath());
});
```

`composedPath()`はイベントが通った経路を返します。ただしclosedなShadow Rootの内部は外部の経路から隠されます。

retargetingは、利用側が`input`や`button`といった内部構造へ依存するのを防ぎます。公開イベントはホストの出来事として設計してください。

## プロパティ設定に反応してイベントを返さない

外部が`item.completed = true`と設定した直後に`task-toggle`を発行すると、設定した側へ同じ情報を返すだけです。双方向バインディングでは循環更新を起こすこともあります。

イベントを発行するのは、利用者のクリック、内部タイマーの完了、ネットワーク結果の受信など、コンポーネント内部で新しい事実が発生したときです。外部から渡された値を表示へ反映するだけなら通知は要りません。

## 取り消せる操作はcancelableにする

削除のように外部が止めたい操作は、確定前のイベントを取り消せるようにします。

```ts
#requestRemove(): void {
  const task = this.#task;
  if (!task) return;

  const accepted = this.dispatchEvent(
    new CustomEvent('task-remove', {
      bubbles: true,
      composed: true,
      cancelable: true,
      detail: { id: task.id },
    }),
  );

  if (!accepted) {
    this.#showRemoveCanceled();
  }
}
```

利用側は`preventDefault()`で取り消します。

```ts
list.addEventListener('task-remove', (event) => {
  if (hasUnsavedChanges()) {
    event.preventDefault();
  }
});
```

`dispatchEvent()`の戻り値は、cancelableなイベントが取り消された場合に`false`です。イベント名に`before-`を付ける設計もありますが、本書では操作要求そのものをcancelableにします。

## bubbleするイベントなら一覧側でまとめて受け取れる

複数の`<task-item>`を持つ`<task-list>`は、各要素へ個別にリスナーを登録しなくても、祖先でイベントを受け取れます。

```ts
class TaskList extends HTMLElement {
  connectedCallback(): void {
    this.addEventListener('task-toggle', this.#handleToggle);
  }

  #handleToggle = (event: Event): void => {
    if (!(event instanceof CustomEvent)) return;
    console.log(event.detail);
  };
}
```

このイベント委譲は、タスクが後から追加されても機能します。要素の数に比例してリスナーを増やす必要もありません。

属性、プロパティ、Slot、イベントがそろい、Custom Elementのデータの出入りが見えてきました。次章では、Shadow DOM内部とホストをCSSで結ぶ方法を学びます。

## 参考資料

- [Event.composed - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Event/composed)
- [Event - DOM Standard](https://dom.spec.whatwg.org/#interface-event)
