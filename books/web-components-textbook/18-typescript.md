---
title: "TypeScriptとの統合"
---

実行時のCustom Element Registryと、コンパイル時の型情報は別の仕組みです。この一点を押さえておくと、この章の作業はすべて説明が付きます。

`customElements.define('task-item', TaskItem)`を呼ぶと、ブラウザは要素名とクラスをRegistryへ登録します。これは実行時の出来事です。TypeScriptの型システムはこの登録を知らないため、`document.querySelector('task-item')`の戻り値は`Element | null`のままですし、`task-toggle`イベントの`detail`も推論できません。

型宣言は、この隔たりを埋める層です。実行時の機能を増やすものではなく、タグ名・プロパティ・イベントという公開契約を開発ツールへ伝え、利用側の間違いを実行前に見つけやすくします。この章では、公開データ型の定義から`HTMLElementTagNameMap`の拡張、実行時の検証、`.d.ts`の配布まで順に見ていきます。

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

`priority`をUnion型にしていても、`getAttribute()`は任意の文字列を返します。『属性とプロパティ』の章の`toPriority()`のように、DOM境界で正規化します。

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

`tsc -p tsconfig.build.json`でJavaScriptと型宣言を生成できます。`package.json`の`exports`から型の入口を公開する方法は、『npmパッケージ化とドキュメント』の章で扱います。

## まとめ

この章では、Custom Elementの公開契約をTypeScriptの型として表す方法を学びました。

- 実行時のRegistryと型情報は別の仕組みなので、`customElements.define()`だけでは型が付きません。
- `Task`や`TaskToggleDetail`のような公開データ型に名前を付け、実装と利用コードで共有します。
- `HTMLElementTagNameMap`を拡張すると、`querySelector()`の戻り値が具体的なクラスになります。
- `HTMLElementEventMap`の拡張は全要素に影響するため、イベント名へ固有の接頭辞を付けます。
- 型引数はDOMを検証しないので、必須の内部要素は`instanceof`で確認します。
- 属性値は任意の文字列が来るため、DOM境界で正規化します。型は開発時の契約、検証は実行時の境界です。
- `#`で始まるprivate fieldは、型からも実行時からも外部に見えません。
- 配布時は`declaration: true`で`.d.ts`を生成し、JavaScriptと一緒に届けます。

型で確かめられるのは、公開契約に沿った値の形までです。イベント経路やフォーカスといったブラウザ上の振る舞いは、次章の『実ブラウザでのテスト』で確認します。
