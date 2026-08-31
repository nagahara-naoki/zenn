---
title: "タスク管理UIを完成させる"
---

この章のゴールは2つです。`<task-input>`、`<task-filter>`、`<task-list>`、`<task-item>`という4つの要素を`<task-app>`へ統合すること。そして、状態を誰が所有し、要素どうしをどう接続するかを決めることです。

使う実装は、これまでの章で組み立ててきたものです。`<task-input>`は『ElementInternalsとフォーム連携』の章、`<task-item>`は『属性とプロパティ』から『アクセシビリティとフォーカス』までの章で確認した公開契約を、1つのファイルへまとめて使います。

状態は`<task-app>`が所有します。子要素は公開プロパティで表示に必要な状態を受け取り、`CustomEvent`で操作を返します。`<task-item>`は`task`プロパティを受け取り、`task-toggle`と`task-remove`を通知します。`<task-input>`は`value`と`reportValidity()`を公開します。

## 完成時のデータフロー

```mermaid
flowchart TD
  App["task-app: tasksとfilterを所有"]
  Input["task-input"]
  Filter["task-filter"]
  List["task-list"]
  Item["task-item"]

  App -->|tasks property| List
  App -->|filter property| Filter
  List -->|task property| Item
  Input -->|form submit| App
  Filter -->|task-filter-change event| App
  Item -->|task-toggle / task-remove| App
```

子要素はアプリケーションの配列を直接変更しません。操作イベントを受け取った`<task-app>`が新しい配列を作り、子要素の表示用プロパティを更新します。配列を書き換えられる場所が1か所に限られるため、表示と状態がずれる余地が小さくなります。

## Task型とfilter型を共有する

要素どうしがやり取りするデータの形は、1つのファイルに置いて全員で共有します。

```ts:src/task.types.ts
export type TaskPriority = 'low' | 'normal' | 'high';
export type TaskFilterValue = 'all' | 'active' | 'completed';

export interface Task {
  id: string;
  label: string;
  completed: boolean;
  priority: TaskPriority;
  createdAt: string;
}
```

時刻はISO 8601文字列として保存します。この形なら表示時にそのまま`Intl.DateTimeFormat`へ渡せますし、JSONへ保存しても形が変わりません。

## 既出の2要素を完成ファイルへまとめる

先に、これまでの章で育ててきた2つの要素を完成形として置いておきます。

`<task-item>`は、`task`プロパティと`completed`属性から表示を作り、利用者の操作をイベントへ変換します。タスク配列そのものには手を触れません。内部ではネイティブのチェックボックスとボタンを使い、ラベルと補助情報はSlotで受け取ります。

:::details task-item.tsの完成コード
```ts:src/task-item.ts
import type { Task } from './task.types.js';

const itemTemplate = document.createElement('template');
itemTemplate.innerHTML = `
  <style>
    :host { display: block; }
    :host([hidden]) { display: none; }
    .container {
      display: grid;
      grid-template-columns: auto 1fr auto;
      gap: 0.75rem;
      align-items: center;
      padding: 0.75rem;
      border: 1px solid var(--task-item-border-color, #d7dce5);
      border-radius: var(--task-item-radius, 0.75rem);
      background: var(--task-item-background-color, white);
    }
    :host([completed]) .label { text-decoration: line-through; }
    .content { display: grid; gap: 0.25rem; }
  </style>
  <article class="container" part="container">
    <input type="checkbox" aria-label="完了状態を切り替える">
    <span class="content">
      <span class="label"><slot name="label"></slot></span>
      <small><slot name="meta"></slot></small>
    </span>
    <button type="button" part="remove-button">削除</button>
  </article>
`;

export class TaskItem extends HTMLElement {
  static observedAttributes = ['completed'];

  #task?: Task;
  #checkbox: HTMLInputElement;
  #removeButton: HTMLButtonElement;

  constructor() {
    super();
    const root = this.attachShadow({ mode: 'open' });
    root.append(itemTemplate.content.cloneNode(true));

    const checkbox = root.querySelector('input');
    const removeButton = root.querySelector('button');
    if (!(checkbox instanceof HTMLInputElement)) {
      throw new Error('checkboxが見つかりません');
    }
    if (!(removeButton instanceof HTMLButtonElement)) {
      throw new Error('remove buttonが見つかりません');
    }

    this.#checkbox = checkbox;
    this.#removeButton = removeButton;
  }

  connectedCallback(): void {
    this.#checkbox.addEventListener('change', this.#handleToggle);
    this.#removeButton.addEventListener('click', this.#handleRemove);
    this.#render();
  }

  disconnectedCallback(): void {
    this.#checkbox.removeEventListener('change', this.#handleToggle);
    this.#removeButton.removeEventListener('click', this.#handleRemove);
  }

  attributeChangedCallback(): void {
    this.#render();
  }

  get task(): Task | undefined {
    return this.#task;
  }

  set task(value: Task | undefined) {
    this.#task = value;
    this.completed = value?.completed ?? false;
    this.#render();
  }

  get completed(): boolean {
    return this.hasAttribute('completed');
  }

  set completed(value: boolean) {
    this.toggleAttribute('completed', value);
  }

  #handleToggle = (): void => {
    if (!this.#task) return;

    this.dispatchEvent(
      new CustomEvent('task-toggle', {
        bubbles: true,
        composed: true,
        detail: {
          id: this.#task.id,
          completed: this.#checkbox.checked,
        },
      }),
    );
  };

  #handleRemove = (): void => {
    if (!this.#task) return;

    this.dispatchEvent(
      new CustomEvent('task-remove', {
        bubbles: true,
        composed: true,
        cancelable: true,
        detail: { id: this.#task.id },
      }),
    );
  };

  #render(): void {
    this.#checkbox.checked = this.completed;
  }
}
```
:::

`<task-input>`は、外側のフォームから1つの入力要素として見えるようにします。内部入力の値、フォームへ送る値、制約検証の結果は、いずれも同じ状態から更新します。DOM上を移動して再接続されたときも、入力途中の値は初期値へ戻しません。

:::details task-input.tsの完成コード
```ts:src/task-input.ts
const inputTemplate = document.createElement('template');
inputTemplate.innerHTML = `
  <style>
    :host { display: block; }
    :host([hidden]) { display: none; }
    label { display: grid; gap: 0.35rem; }
    #error { color: #b42318; }
  </style>
  <label>
    <span><slot name="label">タスク名</slot></span>
    <input type="text" part="input" aria-describedby="error">
  </label>
  <p id="error" part="error" aria-live="polite"></p>
`;

export class TaskInput extends HTMLElement {
  static formAssociated = true;
  static observedAttributes = ['required'];

  #internals: ElementInternals;
  #input: HTMLInputElement;
  #error: HTMLParagraphElement;
  #defaultValue = '';
  #initialized = false;

  constructor() {
    super();
    this.#internals = this.attachInternals();

    const root = this.attachShadow({ mode: 'open' });
    root.append(inputTemplate.content.cloneNode(true));

    const input = root.querySelector('input');
    const error = root.querySelector('#error');
    if (!(input instanceof HTMLInputElement)) {
      throw new Error('inputが見つかりません');
    }
    if (!(error instanceof HTMLParagraphElement)) {
      throw new Error('errorが見つかりません');
    }

    this.#input = input;
    this.#error = error;
  }

  connectedCallback(): void {
    if (!this.#initialized) {
      this.#defaultValue = this.getAttribute('value') ?? '';
      this.value = this.#defaultValue;
      this.#initialized = true;
    }

    this.#input.required = this.required;
    this.#input.addEventListener('input', this.#handleInput);
    this.#validate();
  }

  disconnectedCallback(): void {
    this.#input.removeEventListener('input', this.#handleInput);
  }

  attributeChangedCallback(name: string): void {
    if (name !== 'required') return;
    this.#input.required = this.required;
    this.#validate();
  }

  get value(): string {
    return this.#input.value;
  }

  set value(value: string) {
    this.#input.value = value;
    this.#internals.setFormValue(value);
    this.#validate();
  }

  get required(): boolean {
    return this.hasAttribute('required');
  }

  set required(value: boolean) {
    this.toggleAttribute('required', value);
  }

  get form(): HTMLFormElement | null {
    return this.#internals.form;
  }

  get labels(): NodeList {
    return this.#internals.labels;
  }

  checkValidity(): boolean {
    return this.#internals.checkValidity();
  }

  reportValidity(): boolean {
    return this.#internals.reportValidity();
  }

  focus(options?: FocusOptions): void {
    this.#input.focus(options);
  }

  formDisabledCallback(disabled: boolean): void {
    this.#input.disabled = disabled;
  }

  formResetCallback(): void {
    this.value = this.#defaultValue;
  }

  formStateRestoreCallback(
    state: string | File | FormData | null,
  ): void {
    if (typeof state === 'string') this.value = state;
  }

  #handleInput = (): void => {
    this.#internals.setFormValue(this.#input.value);
    this.#validate();
    this.dispatchEvent(
      new Event('input', { bubbles: true, composed: true }),
    );
  };

  #validate(): void {
    const missing = this.required && this.value.trim() === '';
    if (missing) {
      const message = 'タスク名を入力してください';
      this.#internals.setValidity(
        { valueMissing: true },
        message,
        this.#input,
      );
      this.#error.textContent = message;
      return;
    }

    this.#internals.setValidity({});
    this.#error.textContent = '';
  }
}
```
:::

## task-filterは選択をイベントで通知する

`<task-filter>`は表示中の絞り込み条件を持ちますが、タスク配列は知りません。ボタンが押されたら`task-filter-change`を発火し、実際の絞り込みは`<task-app>`へ任せます。

```ts:src/task-filter.ts
import type { TaskFilterValue } from './task.types.js';

const filterTemplate = document.createElement('template');
filterTemplate.innerHTML = `
  <style>
    :host { display: inline-flex; gap: 0.25rem; }
    :host([hidden]) { display: none; }
    button[aria-pressed='true'] {
      color: white;
      background: var(--task-accent-color, #2563eb);
    }
  </style>
  <button type="button" data-filter="all">すべて</button>
  <button type="button" data-filter="active">未完了</button>
  <button type="button" data-filter="completed">完了済み</button>
`;

export class TaskFilter extends HTMLElement {
  #root: ShadowRoot;
  #value: TaskFilterValue = 'all';

  constructor() {
    super();
    this.#root = this.attachShadow({ mode: 'open' });
    this.#root.append(filterTemplate.content.cloneNode(true));
    this.#root.addEventListener('click', this.#handleClick);
    this.#root.addEventListener('keydown', this.#handleKeydown);
  }

  connectedCallback(): void {
    this.#render();
  }

  get value(): TaskFilterValue {
    return this.#value;
  }

  set value(value: TaskFilterValue) {
    this.#value = value;
    this.#render();
  }

  #handleClick = (event: Event): void => {
    const button = (event.target as Element).closest<HTMLButtonElement>(
      'button[data-filter]',
    );
    if (!button) return;

    const value = button.dataset.filter as TaskFilterValue;
    this.#select(value);
  };

  #handleKeydown = (event: Event): void => {
    if (!(event instanceof KeyboardEvent)) return;
    if (event.key !== 'ArrowLeft' && event.key !== 'ArrowRight') return;

    event.preventDefault();
    const buttons = [
      ...this.#root.querySelectorAll<HTMLButtonElement>('button'),
    ];
    const current = buttons.indexOf(
      this.#root.activeElement as HTMLButtonElement,
    );
    const offset = event.key === 'ArrowRight' ? 1 : -1;
    const next = (current + offset + buttons.length) % buttons.length;
    buttons.forEach((button, index) => {
      button.tabIndex = index === next ? 0 : -1;
    });
    buttons[next]?.focus();
  };

  #select(value: TaskFilterValue): void {
    if (value === this.#value) return;

    this.#value = value;
    this.#render();
    this.dispatchEvent(
      new CustomEvent<TaskFilterValue>('task-filter-change', {
        bubbles: true,
        composed: true,
        detail: value,
      }),
    );
  }

  #render(): void {
    for (const button of this.#root.querySelectorAll('button')) {
      const selected = button.dataset.filter === this.#value;
      button.setAttribute('aria-pressed', String(selected));
      button.tabIndex = selected ? 0 : -1;
    }
  }
}
```

初回接続時にも`#render()`を呼んで、`aria-pressed`と`tabIndex`を設定しています。矢印キーでの移動は、フォーカスできるボタンを常に1つだけにするroving tabindexの実装です。なお、コードを短くするため`customElements.define()`は省いてあります。登録処理は章末でまとめて行います。

## task-listはidを使って既存要素を再利用する

配列が更新されるたびに全要素を作り直すと、チェックボックスのフォーカスや入力途中の状態が失われます。そこでタスクIDをキーにしたMapを持ち、既存の`<task-item>`を探して使い回します。

```ts:src/task-list.ts
import type { Task } from './task.types.js';
import type { TaskItem } from './task-item.js';

const listTemplate = document.createElement('template');
listTemplate.innerHTML = `
  <style>
    :host { display: grid; gap: 0.5rem; }
    :host([hidden]) { display: none; }
  </style>
  <div id="items" role="list"></div>
  <p id="empty">タスクはありません。</p>
`;

export class TaskList extends HTMLElement {
  #root: ShadowRoot;
  #items: HTMLDivElement;
  #empty: HTMLParagraphElement;
  #tasks: Task[] = [];
  #elements = new Map<string, TaskItem>();

  constructor() {
    super();
    this.#root = this.attachShadow({ mode: 'open' });
    this.#root.append(listTemplate.content.cloneNode(true));

    const items = this.#root.querySelector('#items');
    const empty = this.#root.querySelector('#empty');
    if (!(items instanceof HTMLDivElement)) throw new Error('items missing');
    if (!(empty instanceof HTMLParagraphElement)) throw new Error('empty missing');
    this.#items = items;
    this.#empty = empty;
  }

  get tasks(): readonly Task[] {
    return this.#tasks;
  }

  set tasks(value: readonly Task[]) {
    this.#tasks = [...value];
    this.#render();
  }

  #render(): void {
    const activeIds = new Set(this.#tasks.map((task) => task.id));

    for (const [id, element] of this.#elements) {
      if (!activeIds.has(id)) {
        element.remove();
        this.#elements.delete(id);
      }
    }

    for (const task of this.#tasks) {
      let item = this.#elements.get(task.id);

      if (!item) {
        item = document.createElement('task-item');
        item.setAttribute('role', 'listitem');
        this.#elements.set(task.id, item);
      }

      item.task = task;
      item.completed = task.completed;

      let label = item.querySelector('[slot="label"]');
      if (!label) {
        label = document.createElement('span');
        label.setAttribute('slot', 'label');
        item.append(label);
      }
      label.textContent = task.label;

      this.#items.append(item);
    }

    this.#empty.hidden = this.#tasks.length > 0;
  }
}
```

既存の要素をもう一度`append()`すれば、順序を並べ替えられます。ただしこの移動は切断と再接続として扱われます。『ライフサイクルコールバック』の章で説明したとおり、`<task-item>`は再接続されても状態を失わないように実装しておきます。対象環境が`moveBefore()`に対応していれば、状態を保ったままの移動へ置き換えられます。

## task-appが状態を所有する

タスク配列と絞り込み条件を持つのは`<task-app>`だけです。子要素から上がってきたイベントを受け取り、新しい状態を作り、子要素のプロパティへ流し込みます。

```ts:src/task-app.ts
import type { Task, TaskFilterValue } from './task.types.js';
import type { TaskFilter } from './task-filter.js';
import type { TaskInput } from './task-input.js';
import type { TaskList } from './task-list.js';

const appTemplate = document.createElement('template');
appTemplate.innerHTML = `
  <style>
    :host {
      display: grid;
      gap: 1rem;
      max-inline-size: 44rem;
      margin-inline: auto;
      font-family: system-ui, sans-serif;
    }
    :host([hidden]) { display: none; }
    form { display: flex; gap: 0.5rem; align-items: end; }
    task-input { flex: 1; }
  </style>

  <header><slot name="heading"><h1>タスク</h1></slot></header>
  <form>
    <task-input name="label" required></task-input>
    <button type="submit">追加</button>
  </form>
  <task-filter></task-filter>
  <task-list></task-list>
  <p id="status" aria-live="polite"></p>
`;

export class TaskApp extends HTMLElement {
  #root: ShadowRoot;
  #input: TaskInput;
  #filter: TaskFilter;
  #list: TaskList;
  #status: HTMLParagraphElement;
  #tasks: Task[] = [];
  #filterValue: TaskFilterValue = 'all';

  constructor() {
    super();
    this.#root = this.attachShadow({ mode: 'open' });
    this.#root.append(appTemplate.content.cloneNode(true));

    const input = this.#root.querySelector('task-input');
    const filter = this.#root.querySelector('task-filter');
    const list = this.#root.querySelector('task-list');
    const status = this.#root.querySelector('#status');
    if (!input || !filter || !list || !(status instanceof HTMLParagraphElement)) {
      throw new Error('task-appの初期化に失敗しました');
    }

    this.#input = input;
    this.#filter = filter;
    this.#list = list;
    this.#status = status;

    this.#root.querySelector('form')?.addEventListener(
      'submit',
      this.#handleSubmit,
    );
    this.addEventListener('task-filter-change', this.#handleFilter);
    this.addEventListener('task-toggle', this.#handleToggle);
    this.addEventListener('task-remove', this.#handleRemove);
  }

  #handleSubmit = (event: Event): void => {
    event.preventDefault();
    if (!this.#input.reportValidity()) return;

    const label = this.#input.value.trim();
    if (!label) return;

    const task: Task = {
      id: crypto.randomUUID(),
      label,
      completed: false,
      priority: 'normal',
      createdAt: new Date().toISOString(),
    };

    this.#tasks = [task, ...this.#tasks];
    this.#input.value = '';
    this.#status.textContent = `「${label}」を追加しました`;
    this.#render();
    this.#input.focus();
  };

  #handleFilter = (event: Event): void => {
    if (!(event instanceof CustomEvent)) return;
    this.#filterValue = event.detail as TaskFilterValue;
    this.#render();
  };

  #handleToggle = (event: Event): void => {
    if (!(event instanceof CustomEvent)) return;
    const { id, completed } = event.detail as {
      id: string;
      completed: boolean;
    };

    this.#tasks = this.#tasks.map((task) =>
      task.id === id ? { ...task, completed } : task,
    );
    this.#render();
  };

  #handleRemove = (event: Event): void => {
    if (!(event instanceof CustomEvent)) return;
    const { id } = event.detail as { id: string };
    queueMicrotask(() => {
      if (event.defaultPrevented) {
        this.#status.textContent = '削除を取り消しました';
        return;
      }

      this.#tasks = this.#tasks.filter((task) => task.id !== id);
      this.#status.textContent = 'タスクを削除しました';
      this.#render();
    });
  };

  #render(): void {
    this.#filter.value = this.#filterValue;
    this.#list.tasks = this.#tasks.filter((task) => {
      if (this.#filterValue === 'active') return !task.completed;
      if (this.#filterValue === 'completed') return task.completed;
      return true;
    });
  }
}
```

`task-remove`は、削除の通知ではなく、取り消せる操作要求です。`<task-app>`は状態変更を`queueMicrotask()`まで遅らせ、伝播の途中で外部が`preventDefault()`を呼んだかどうかを確認します。取り消されていなければ、状態を所有する`<task-app>`が削除を確定します。削除の可否を外部が決められる一方、配列を書き換えるのはあくまで所有者だけです。

## 依存順に一度だけ登録する

`<task-app>`のコンストラクターは、内部で子Custom Elementsを生成します。そのため子要素を先に登録し、アプリケーションを最後に登録します。`customElements.get()`で確認してから登録すれば、モジュールが二重に読み込まれても例外になりません。

```ts:src/register.ts
import { TaskApp } from './task-app.js';
import { TaskFilter } from './task-filter.js';
import { TaskInput } from './task-input.js';
import { TaskItem } from './task-item.js';
import { TaskList } from './task-list.js';

const definitions = [
  ['task-input', TaskInput],
  ['task-filter', TaskFilter],
  ['task-item', TaskItem],
  ['task-list', TaskList],
  ['task-app', TaskApp],
] as const;

for (const [name, constructor] of definitions) {
  if (!customElements.get(name)) {
    customElements.define(name, constructor);
  }
}
```

登録処理をまとめておくと、ページのHTMLは短いままで済みます。

```html:index.html
<!doctype html>
<html lang="ja">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Task Components</title>
    <script type="module" src="/src/register.ts"></script>
  </head>
  <body>
    <task-app>
      <h1 slot="heading">今日のタスク</h1>
    </task-app>
  </body>
</html>
```

## 完成後に確認すること

実装が動いたら、次の項目を順に確認します。

- 定義前のLight DOMが読め、定義後に正しくアップグレードされる
- タスク名を空のまま送信できない
- Enterで追加し、追加後は入力欄へフォーカスが戻る
- チェック操作が`task-toggle`としてShadow DOMを越える
- フィルターを矢印キーで移動できる
- 削除後に状態と一覧DOMが一致する
- 強制カラーモードと200%拡大でも操作できる

『実ブラウザでのテスト』と『フレームワークとの連携』の章まで読んでいれば、次も確認できます。

- Chromium、Firefox、WebKitのPlaywrightテストが通る
- React、Vue、Angularのいずれかから、`<task-item>`の`task`プロパティと操作イベントを扱える

永続化を加えたくなったら、`<task-app>`の内部へ`fetch()`を書き足す前に境界を考えます。状態の所有を外部へ移すなら、`tasks`プロパティと`tasks-change`イベントを公開契約として先に追加します。そのうえで保存処理を外へ置けば、REST API、IndexedDB、テスト用のメモリー実装を差し替えられます。

## 公開契約という一貫した考え方

本書は、`HTMLElement`を継承する最小クラスから始めました。そこから属性とプロパティ、ライフサイクル、Shadow DOM、Slot、イベント、スタイル、フォーム、テスト、配布へと範囲を広げてきました。

中心にある考え方は、どの章でも同じです。利用者が触るHTML、DOMプロパティ、イベント、Slot、スタイルAPIを先に設計し、内部実装をその契約の内側へ閉じる。この順序を守れば、内部をどう書き換えても利用側は壊れません。

ネイティブDOM APIで実装しても、Litへ移行しても、ReactやAngularから利用しても、契約が残るかぎり部品は使い続けられます。Web Componentsはフレームワークを不要にする技術ではなく、Webという共通基盤の上で再利用可能なUIを提供するための技術です。

## まとめ

この章では、4つの要素を`<task-app>`へ統合しました。

- 状態を所有するのは`<task-app>`だけで、子要素はプロパティを受け取りイベントを返します。
- `<task-list>`はタスクIDで既存の`<task-item>`を探し、要素を作り直さずに再利用します。
- `task-remove`はcancelableな操作要求として扱い、`preventDefault()`を確認してから削除を確定します。
- 登録は依存順に、子要素から`<task-app>`の順で一度だけ行います。
- 永続化のような機能を足すときも、まず公開契約を決めてから内部実装を選びます。

次章では、本書に出てきた用語の関係を整理し、困りごとから読む章を引ける索引をまとめます。
