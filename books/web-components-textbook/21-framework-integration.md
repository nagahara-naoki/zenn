---
title: "React・Vue・AngularとはDOMの契約でつなぐ"
---

適切に設計したCustom Elementは、フレームワーク専用のラッパーがなくても利用できます。短い値は属性、オブジェクトはプロパティ、内側からの通知はCustom Event、表示内容はSlotという契約を守っているためです。

フレームワークごとに違うのは、そのDOM APIをテンプレートから呼ぶ記法です。

統合時に見るべき点は、タグを表示できるかだけではありません。文字列属性とオブジェクトプロパティを区別できるか、Custom Eventを購読して解除できるか、TypeScriptへタグ名を知らせられるかを確認します。ここが曖昧だと、HTMLには表示できてもデータ更新で行き詰まります。

Web Components側が標準のDOM契約を守っていれば、フレームワーク専用コードは薄い接続層に留まります。逆に、属性へ巨大なJSONを詰めたり、内部メソッドを直接呼ばせたりすると、接続層が複雑になります。

## 登録処理をアプリケーションの入口で読み込む

どのフレームワークでも、要素を描画する前に定義モジュールを読み込みます。

```ts
import '@example/task-elements/register';
```

遅延読み込みする場合は、定義前の表示を用意し、Custom Elementのプロパティアップグレードにも対応します。第6章の`#upgradeProperty()`がここで効きます。

## React 19ではプロパティとCustom Eventを扱える

React 19は、Custom Elementのインスタンスに存在する名前をDOMプロパティとして設定します。`task`のようなオブジェクトも、Custom Element側にgetterまたはsetterがあれば渡せます。

```tsx
import { useState } from 'react';
import '@example/task-elements/register';
import type { Task, TaskToggleEvent } from '@example/task-elements';

export function TaskScreen() {
  const [task, setTask] = useState<Task>({
    id: 'task-1',
    label: '原稿をレビューする',
    completed: false,
    priority: 'normal',
  });

  function handleToggle(event: TaskToggleEvent) {
    setTask((current) => ({
      ...current,
      completed: event.detail.completed,
    }));
  }

  return (
    <task-item task={task} ontask-toggle={handleToggle}>
      <span slot="label">{task.label}</span>
    </task-item>
  );
}
```

Custom Event名にハイフンを含める場合、JSX属性も`on`に続けて同じ名前を使います。チームのReactや型定義の構成で扱いにくい場合は、`ref`から標準の`addEventListener()`を使うラッパーを1つ用意すると明示的です。

```tsx
import { useEffect, useRef } from 'react';

const ref = useRef<TaskItem>(null);

useEffect(() => {
  const item = ref.current;
  if (!item) return;

  const controller = new AbortController();
  item.addEventListener('task-toggle', handleToggle, {
    signal: controller.signal,
  });

  return () => controller.abort();
}, []);
```

TypeScriptへCustom Elementを知らせるには、ReactのJSX型を拡張します。

```ts:src/task-elements.react.d.ts
import type * as React from 'react';
import type { Task, TaskItem, TaskToggleEvent } from '@example/task-elements';

declare module 'react' {
  namespace JSX {
    interface IntrinsicElements {
      'task-item': React.DetailedHTMLProps<
        React.HTMLAttributes<TaskItem>,
        TaskItem
      > & {
        task?: Task;
        'ontask-toggle'?: (event: TaskToggleEvent) => void;
      };
    }
  }
}
```

## VueではCustom Elementとしてコンパイルする設定を入れる

Vueのテンプレートコンパイラーは、未知のタグをVueコンポーネントとして解決しようとします。ハイフンを含むタグをCustom Elementとして扱う設定を追加します。

```ts:vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
  plugins: [
    vue({
      template: {
        compilerOptions: {
          isCustomElement: (tag) => tag.startsWith('task-'),
        },
      },
    }),
  ],
});
```

テンプレートでは、通常のプロパティbindingとイベント購読を使います。

```vue
<script setup lang="ts">
import { ref } from 'vue';
import '@example/task-elements/register';
import type { Task, TaskToggleEvent } from '@example/task-elements';

const task = ref<Task>({
  id: 'task-1',
  label: '原稿をレビューする',
  completed: false,
  priority: 'normal',
});

function handleToggle(event: TaskToggleEvent) {
  task.value = {
    ...task.value,
    completed: event.detail.completed,
  };
}
</script>

<template>
  <task-item :task="task" @task-toggle="handleToggle">
    <span slot="label">{{ task.label }}</span>
  </task-item>
</template>
```

Vueが値を属性へ渡してしまう要素では、`.prop`修飾子でDOMプロパティを明示できます。

```vue
<task-item :task.prop="task"></task-item>
```

Custom Element側が`task`プロパティを定義していれば、通常はVueが自動判定します。

## AngularではCUSTOM_ELEMENTS_SCHEMAを宣言する

AngularのStandalone Componentから未知のCustom Elementを使う場合、`CUSTOM_ELEMENTS_SCHEMA`を追加します。

```ts
import {
  Component,
  CUSTOM_ELEMENTS_SCHEMA,
  signal,
} from '@angular/core';
import '@example/task-elements/register';
import type { Task, TaskToggleEvent } from '@example/task-elements';

@Component({
  selector: 'app-task-screen',
  schemas: [CUSTOM_ELEMENTS_SCHEMA],
  template: `
    <task-item
      [task]="task()"
      (task-toggle)="handleToggle($event)"
    >
      <span slot="label">{{ task().label }}</span>
    </task-item>
  `,
})
export class TaskScreen {
  readonly task = signal<Task>({
    id: 'task-1',
    label: '原稿をレビューする',
    completed: false,
    priority: 'normal',
  });

  handleToggle(event: Event): void {
    const toggle = event as TaskToggleEvent;
    this.task.update((task) => ({
      ...task,
      completed: toggle.detail.completed,
    }));
  }
}
```

`[task]`はDOMプロパティへ値を渡し、`(task-toggle)`はCustom Eventを受け取ります。Angularのテンプレート型検査がイベントdetailを推論しない場合は、ハンドラー境界で型を確認します。

## 属性とプロパティの違いが連携品質を決める

次のAPIはどのフレームワークでも扱いやすくなります。

- クラスのprototype上に公開プロパティのgetter/setterがある
- 真偽属性が存在の有無で動く
- オブジェクトをJSON属性へ要求しない
- イベントが`bubbles`と`composed`の意図を明示する
- 複雑な表示内容を標準Slotで受け取る

フレームワーク別コードを増やす前に、Custom Element自身がDOM要素として自然かを見直してください。

Web Componentsはフレームワークの状態管理を置き換えません。上の例でも、状態の所有者はReactの`useState`、Vueの`ref`、Angularの`signal`です。Custom ElementはUI境界として値を受け取り、操作を通知します。

サーバーから返すHTMLにもShadow DOMを含めたい場合は、第22章のDeclarative Shadow DOMへ進んでください。

## 参考資料

- [React DOM: Custom HTML elements](https://react.dev/reference/react-dom/components#custom-html-elements)
- [Vue and Web Components](https://vuejs.org/guide/extras/web-components.html)
- [Adding event listeners - Angular](https://angular.dev/guide/templates/event-listeners)
