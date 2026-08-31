---
title: "属性とプロパティ"
---

Custom Elementへ値を渡す入口は、HTML属性とDOMプロパティの2つです。属性はHTMLに書ける文字列で、プロパティはメモリ上にある要素オブジェクトの現在値です。名前が同じでも、扱える値と保存される場所が違います。

属性はHTML文書に書ける情報です。文字列として直列化され、ページのソース、開発者ツール、`getAttribute()`から確認できます。プロパティは要素オブジェクトのフィールドなので、HTMLへ書き戻せないオブジェクトや関数もそのまま保持できます。

組み込みの`input`にも両方があります。HTMLの`value`属性は初期値を表し、利用者が入力した現在値は`input.value`プロパティにあります。Custom Elementでも、宣言時の設定と実行中の状態を同じ名前で扱うなら、この2層を意識して設計します。

```html
<task-item completed priority="high"></task-item>
```

```ts
const item = document.querySelector('task-item');
item.completed = true;
item.task = { id: 'task-1', label: '原稿を書く' };
```

属性はHTMLに文字列として残ります。プロパティはJavaScriptオブジェクト上の値なので、真偽値、配列、関数、オブジェクトもそのまま保持できます。

## 値の性質から入口を選ぶ

属性に向くのは、HTMLだけで意味が読み取れ、文字列へ直しても情報を失わない値です。

| 値 | 推奨する入口 | 理由 |
|---|---|---|
| `completed` | 真偽属性 | HTMLから状態が分かる |
| `priority="high"` | 属性 | 短い列挙値として表せる |
| `task.label` | Slotまたは属性 | 表示する短い文字列 |
| `task`オブジェクト | プロパティ | 文字列化すると型を失う |
| タスク配列 | プロパティ | 頻繁なJSON変換を避けられる |
| コールバック | イベント | DOMの通知モデルと合う |

オブジェクトをJSON属性へ詰め込む方法は、引用符のエスケープと更新コストを増やします。JavaScriptから渡す値なら、プロパティのまま受け取るほうが自然です。

## 真偽属性は存在すればtrueになる

HTMLの`disabled`や`required`と同じく、真偽属性は値の文字列を読みません。属性が存在すれば`true`です。

```html
<task-item completed></task-item>
<task-item completed=""></task-item>
<task-item completed="false"></task-item>
```

上の3つはすべて`completed`が存在するため、同じ状態です。`false`に戻すには属性を削除します。

```ts
item.toggleAttribute('completed', false);
```

プロパティからも扱いやすいように、getterとsetterを用意します。

```ts
class TaskItem extends HTMLElement {
  static observedAttributes = ['completed'];

  get completed(): boolean {
    return this.hasAttribute('completed');
  }

  set completed(value: boolean) {
    this.toggleAttribute('completed', Boolean(value));
  }

  attributeChangedCallback(
    name: string,
    oldValue: string | null,
    newValue: string | null,
  ): void {
    if (name === 'completed' && oldValue !== newValue) {
      this.#updateCompleted();
    }
  }

  #updateCompleted(): void {
    // 表示更新は『状態とレンダリング』の章で実装する
  }
}
```

setterは属性を変更し、属性変更のコールバックが表示を更新します。更新経路をひとつにすると、属性から変更した場合とプロパティから変更した場合で表示がずれません。

## 列挙属性は不正な値を既定値へ戻す

`priority`は`low`、`normal`、`high`の3種類だけを受け付けるとします。

```ts
type TaskPriority = 'low' | 'normal' | 'high';

function toPriority(value: string | null): TaskPriority {
  if (value === 'low' || value === 'high') return value;
  return 'normal';
}

class TaskItem extends HTMLElement {
  get priority(): TaskPriority {
    return toPriority(this.getAttribute('priority'));
  }

  set priority(value: TaskPriority) {
    this.setAttribute('priority', value);
  }
}
```

HTMLは外部から編集されます。型定義だけで入力を信頼せず、属性を読む境界で実行時に正規化します。

数値属性も同じです。`Number()`の結果が`NaN`でないか、範囲内かを確かめてから使います。

## 複雑な値はプロパティだけで受け取る

タスク全体はオブジェクトとして渡します。

```ts
export interface Task {
  id: string;
  label: string;
  completed: boolean;
}

class TaskItem extends HTMLElement {
  #task?: Task;

  get task(): Task | undefined {
    return this.#task;
  }

  set task(value: Task | undefined) {
    this.#task = value;
    this.#requestUpdate();
  }

  #requestUpdate(): void {}
}
```

受け取ったオブジェクトを内部で書き換えると、呼び出し側の状態まで変わります。コンポーネントから変更を求めるときは、元オブジェクトを直接変更せずイベントで意思を通知します。

## 定義前に設定されたプロパティを拾い直す

Custom ElementのJavaScriptが遅れて読み込まれると、定義前の要素へプロパティが設定されることがあります。

```ts
const item = document.querySelector('task-item');
item.task = task;

await import('./task-item');
```

定義前に作られた`task`は要素自身のプロパティになります。アップグレード後も、クラスのprototypeにあるsetterを隠してしまいます。接続時に値をいったん削除し、setterへ渡し直します。

```ts
class TaskItem extends HTMLElement {
  connectedCallback(): void {
    this.#upgradeProperty('task');
  }

  #upgradeProperty(name: keyof TaskItem): void {
    if (!Object.prototype.hasOwnProperty.call(this, name)) return;

    const value = this[name];
    delete (this as Partial<TaskItem>)[name];
    this[name] = value as never;
  }
}
```

遅延アップグレードに対応しておくと、HTMLとJavaScriptの読み込み順にAPIが左右されにくくなります。

## 反映する属性を選ぶ

プロパティの変更を属性へ戻す処理をreflectionと呼びます。すべての値を反映する必要はありません。

`completed`のようにCSSセレクター、フォーム、開発者ツールから状態を確認したい値は反映すると便利です。一時的な検索結果や大きな配列は、属性へ反映しても利用価値がありません。

属性は公開APIです。追加する前に「HTMLへ残ることで誰が得をするか」を考えると、無駄な同期を減らせます。

## まとめ

この章では、Custom Elementの値の入口となる属性とプロパティを学びました。

- 属性はHTMLに書ける文字列、プロパティは要素オブジェクトの現在値です。
- 文字列で表せて、HTMLから状態が読めるとうれしい値は属性に向いています。
- オブジェクトや配列、関数はプロパティで受け取ります。
- 真偽属性は値の内容を見ず、属性が存在すれば`true`になります。
- 列挙値や数値は、属性を読む境界で実行時に正規化します。
- 定義前に設定されたプロパティは、接続時に削除してsetterへ渡し直します。
- プロパティを属性へ戻すreflectionは、CSSや開発者ツールから見たい値だけに絞ります。

次章では、受け取った値を内部状態としてまとめ、DOMを壊さずに表示を更新する方法を実装します。
