---
title: "CustomEventとイベント伝播"
---

CustomEventとは、独自の名前と任意のデータ（`detail`）を持たせて発火できるDOMイベントです。Custom Elementは、内部で起きた出来事をCustomEventとして外へ通知します。

外から内へ値を渡す手段が属性とプロパティなら、内から外へ知らせる手段がイベントです。DOM標準の通知モデルに合わせておくと、Vanilla JavaScriptからでもフレームワークからでも、同じ`addEventListener()`で受け取れます。

コールバック関数をプロパティで渡す方法でも通知はできます。ただしDOMには、複数の祖先が同じ出来事を受け取れる伝播の仕組みが最初から備わっています。イベントを使えば、一覧側で子要素の操作をまとめて扱えます。

通知するのは、内部DOMで発生した低水準の操作そのものではありません。コンポーネントとして外へ伝えたい出来事です。内部ボタンの`click`をそのまま公開せず、「タスクの完了状態を変えたい」という`task-toggle`へ翻訳します。この翻訳があれば、内部をボタンから別のUIへ変えても公開契約を保てます。

## detailに入れる情報

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

第1引数がイベント名、第2引数が設定です。`detail`へ渡した値は、受け取り側の`event.detail`から読めます。

```ts
document.addEventListener('task-toggle', (event) => {
  if (!(event instanceof CustomEvent)) return;

  console.log(event.detail.id);
  console.log(event.detail.completed);
});
```

`detail`へ内部要素の参照を入れると、利用側が実装へ依存してしまいます。公開するのは、状態更新に必要な識別子と値だけにします。

## bubblesとcomposed

イベントがどこまで届くかは、2つの設定で決まります。それぞれ制御する境界が違います。

`bubbles`は、発生元から親方向へさかのぼるかどうかを決めます。`composed`は、Shadow DOMの境界を越えて外へ出られるかどうかを決めます。両方を`true`にして初めて、Shadow Root内部で発火したイベントがページ側の祖先へ届きます。

| `bubbles` | `composed` | 結果 |
|---|---|---|
| `false` | `false` | 発生元だけで処理される |
| `true` | `false` | 同じShadow Tree内を伝播する |
| `false` | `true` | 境界を越えられるが通常のbubbleはしない |
| `true` | `true` | Shadow DOMを越えて祖先へ伝播する |

アプリケーションへ公開する操作イベントには、通常`bubbles: true`と`composed: true`を指定します。内部だけで使うイベントは`composed: false`のままにして、実装詳細を境界の内側へ留めます。

なお、ブラウザ組み込みのイベントにも同じ区別があります。`click`や`focus`は`composed: true`ですが、`change`や`submit`は`composed: false`です。内部の`input`が発火した`change`はホストの外へ出ないため、公開したい出来事は自分でCustomEventへ翻訳することになります。

## retargetingとevent.target

境界を越えたイベントには、もう1つ仕掛けがあります。Shadow DOM内部の`input`から出たイベントを外部で受け取ると、`event.target`は内部の`input`ではなく`<task-item>`になります。ブラウザが発生元をホストへ置き換えるためで、この処理をretargetingと呼びます。

```ts
document.addEventListener('click', (event) => {
  console.log(event.target); // <task-item>になる場合がある
  console.log(event.composedPath());
});
```

`composedPath()`は、イベントが実際に通った経路を配列で返します。openなShadow Rootなら内部の要素も含まれます。closedなShadow Rootの内部は、外部から見た経路には現れません。

retargetingがあるおかげで、利用側が`input`や`button`といった内部構造へ依存せずに済みます。公開イベントは、ホストの出来事として設計してください。

## プロパティ設定に反応させない

外部が`item.completed = true`と設定した直後に`task-toggle`を発行すると、設定した側へ同じ情報を返すだけになります。双方向バインディングでは循環更新の原因にもなります。

イベントを発行するのは、利用者のクリック、内部タイマーの完了、ネットワーク結果の受信など、コンポーネント内部で新しい事実が生まれたときです。外部から渡された値を表示へ反映するだけなら、通知は要りません。

## cancelableで操作を止められるようにする

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

`dispatchEvent()`の戻り値は、cancelableなイベントが取り消されたとき`false`になります。イベント名に`before-`を付ける設計もありますが、本書では操作要求そのものをcancelableにします。

## 一覧側でまとめて受け取る

複数の`<task-item>`を持つ`<task-list>`は、各要素へ個別にリスナーを登録しなくても、祖先の位置でイベントを受け取れます。

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

## まとめ

この章では、CustomEventとイベント伝播を学びました。

- CustomEventは、独自の名前と`detail`を持つDOMイベントです。
- `detail`には、状態更新に必要な識別子と値だけを入れます。
- `bubbles`は親方向への伝播、`composed`はShadow DOM境界の通過を制御します。
- 境界を越えたイベントの`event.target`はホストへ置き換わります（retargeting）。経路は`composedPath()`で確認できます。
- 外部から設定された値をそのまま通知し返すと、循環更新の原因になります。
- 取り消せる操作は`cancelable: true`にし、`dispatchEvent()`の戻り値で判定します。
- bubbleするイベントなら、一覧側でまとめて受け取れます。

属性、プロパティ、Slot、イベントがそろい、Custom Elementのデータの出入りが見えてきました。次章では、Shadow DOM内部とホストをCSSで結ぶ方法を学びます。

## 参考資料

- [Event.composed - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Event/composed)
- [Event - DOM Standard](https://dom.spec.whatwg.org/#interface-event)
