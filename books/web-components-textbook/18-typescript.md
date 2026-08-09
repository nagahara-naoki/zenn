---
title: "TypeScriptで要素名・プロパティ・イベントを型にする"
---

ブラウザはCustom Elementの名前とクラスを実行時に結びつけます。TypeScriptの型システムは、その登録を自動では知りません。公開する要素名、プロパティ、イベントを型宣言に追加すると、利用側の補完と検査が働きます。

実行時のCustom Element Registryと、コンパイル時の型情報は別の仕組みです。`customElements.define()`を呼べばブラウザでは動きますが、それだけでは`document.querySelector('task-item')`の戻り値やイベントの`detail`をTypeScriptが推論できません。

型宣言は、実行時の機能を増やすものではありません。タグ名、プロパティ、イベントという公開契約を開発ツールにも伝え、利用側の間違いを実行前に見つけやすくする層です。

## 公開データ型を先に定義する

```ts:src/task-item.types.ts
export interface Task {
  id: string;
  label: string;
  completed: boolean;
  priority: 'low' | 'normal' | 'high';
}

export interface TaskToggleDetail {
  id: string;
  completed: boolean;
}

export type TaskToggleEvent = CustomEvent<TaskToggleDetail>;
```

イベントの`detail`を匿名オブジェクトのまま各所へ書くと、項目追加時にずれます。公開型へ名前を付け、要素実装と利用コードから共有します。

## クラスの公開プロパティを明示する

```ts:src/task-item.ts
import type { Task, TaskToggleDetail } from './task-item.types.js';

export const taskItemTagName = 'task-item' as const;

export class TaskItem extends HTMLElement {
  #task?: Task;

  get task(): Task | undefined {
    return this.#task;
  }

  set task(value: Task | undefined) {
    this.#task = value;
    this.#requestUpdate();
  }

  #notifyToggle(completed: boolean): void {
    if (!this.#task) return;

    const detail: TaskToggleDetail = {
      id: this.#task.id,
      completed,
    };

    this.dispatchEvent(
      new CustomEvent<TaskToggleDetail>('task-toggle', {
        bubbles: true,
        composed: true,
        detail,
      }),
    );
  }

  #requestUpdate(): void {}
}
```

タグ名を定数にすると、登録、型宣言、テストで同じ文字列を使えます。

## HTMLElementTagNameMapへ要素を登録する

`document.querySelector('task-item')`の戻り値を`TaskItem | null`にするには、グローバルな`HTMLElementTagNameMap`を拡張します。

```ts:src/task-item.global.d.ts
import type { TaskItem } from './task-item.js';
import type { TaskToggleEvent } from './task-item.types.js';

declare global {
  interface HTMLElementTagNameMap {
    'task-item': TaskItem;
  }

  interface HTMLElementEventMap {
    'task-toggle': TaskToggleEvent;
  }
}

export {};
```

利用側で型引数や型アサーションを書かなくても補完されます。

```ts
const item = document.querySelector('task-item');

if (item) {
  item.task = {
    id: 'task-1',
    label: '型定義を書く',
    completed: false,
    priority: 'high',
  };

  item.addEventListener('task-toggle', (event) => {
    console.log(event.detail.completed);
  });
}
```

`HTMLElementEventMap`の拡張はすべてのHTML要素のイベント候補へ追加されます。ライブラリのイベント名には固有の接頭辞を付け、衝突を避けます。より狭い型を求める場合は、`TaskItem`クラスへ`addEventListener()`のoverloadを定義する方法もあります。

## querySelectorの結果は実行時にも確認する

型引数は実際のDOMを検証しません。

```ts
const input = root.querySelector<HTMLInputElement>('.input');
```

セレクターが間違っていても、TypeScriptは`HTMLInputElement | null`として扱います。必須の内部要素は取得直後に確認します。

```ts
const input = root.querySelector('.input');

if (!(input instanceof HTMLInputElement)) {
  throw new Error('.inputがHTMLInputElementではありません');
}
```

エラーを初期化時に出すため、後から`Cannot read properties of null`になるより原因を特定しやすくなります。

## 属性値は実行時の文字列として検証する

TypeScriptはHTMLファイルの属性を保証しません。

```html
<task-item priority="urgent"></task-item>
```

`priority`をUnion型にしていても、`getAttribute()`は任意の文字列を返します。第6章の`toPriority()`のように、DOM境界で正規化します。

型は開発時の契約、検証は実行時の境界です。どちらか一方で代用しません。

## private fieldで内部実装を型から隠す

`#root`や`#render()`のようなprivate fieldは、利用側の型から見えず、実行時にも外部アクセスできません。

```ts
class TaskItem extends HTMLElement {
  #root: ShadowRoot;
  #render(): void {}
}
```

TypeScriptの`private`は主に型検査上の制限です。JavaScriptの`#`は言語レベルのprivate fieldです。公開契約へ含めない実装には`#`を使うと、誤利用を防ぎやすくなります。

## declarationファイルをパッケージへ含める

配布時は、JavaScriptと一緒に`.d.ts`を生成します。

```json:tsconfig.build.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "declaration": true,
    "declarationMap": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

`tsc -p tsconfig.build.json`でJavaScriptと型宣言を生成できます。第20章では`package.json`の`exports`から型の入口を公開します。

TypeScriptで確かめられるのは、公開契約に沿った値の形までです。イベント経路やフォーカスなど、ブラウザ上の振る舞いは第19章のPlaywrightテストで確認します。
