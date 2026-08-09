---
title: "クラスと登録処理を分けてnpmパッケージにする"
---

Custom Elementのモジュールをimportしただけでグローバル登録される設計は手軽です。ただし、利用者が名前を変えたい場合、テストごとに異なるクラスを登録したい場合、複数バージョンを共存させたい場合には制約になります。

配布パッケージでは、要素クラスと登録処理を別の入口として公開します。

この分離が必要なのは、Custom Element Registryがページ単位の共有資源だからです。同じ名前は原則として一度しか登録できず、あとから別のクラスへ置き換えられません。ライブラリをimportしただけで登録すると、利用者が名前や登録時期を選ぶ余地がなくなります。

クラスをexportする入口と、既定名で登録する便利な入口を分ければ、単純な利用と高度な組み込みの両方に対応できます。パッケージ化はファイルを圧縮する作業ではなく、登録方法を含む公開契約を決める作業です。

## 配布用のディレクトリを分ける

```text
task-elements/
├─ src/
│  ├─ task-item.ts
│  ├─ task-item.types.ts
│  ├─ task-input.ts
│  ├─ index.ts
│  └─ register.ts
├─ package.json
└─ tsconfig.build.json
```

`index.ts`はクラスと型をexportします。読み込んでも登録は行いません。

```ts:src/index.ts
export { TaskItem } from './task-item.js';
export { TaskInput } from './task-input.js';
export type {
  Task,
  TaskToggleDetail,
  TaskToggleEvent,
} from './task-item.types.js';
```

`register.ts`だけがグローバルなCustom Element Registryを変更します。

```ts:src/register.ts
import { TaskInput } from './task-input.js';
import { TaskItem } from './task-item.js';

export function registerTaskElements(): void {
  if (!customElements.get('task-item')) {
    customElements.define('task-item', TaskItem);
  }

  if (!customElements.get('task-input')) {
    customElements.define('task-input', TaskInput);
  }
}

registerTaskElements();
```

## exportsで2つの入口を公開する

```json:package.json
{
  "name": "@example/task-elements",
  "version": "1.0.0",
  "type": "module",
  "files": ["dist", "custom-elements.json"],
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./register": {
      "types": "./dist/register.d.ts",
      "import": "./dist/register.js"
    }
  },
  "sideEffects": ["./dist/register.js"],
  "scripts": {
    "build": "tsc -p tsconfig.build.json"
  }
}
```

`sideEffects`へ登録用ファイルを書くと、bundler（複数のモジュールを配布用にまとめるツール）が「未使用のexportしかない」と判断して削除するのを防げます。クラスだけを読みたい利用者は通常の入口を使います。

```ts
import { TaskItem } from '@example/task-elements';

customElements.define('project-task', TaskItem);
```

既定名でまとめて登録したい場合は副作用入口を読みます。

```ts
import '@example/task-elements/register';
```

## TypeScriptでES Modulesと型宣言を生成する

```json:tsconfig.build.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "rootDir": "src",
    "outDir": "dist"
  },
  "include": ["src"]
}
```

NodeNextで相対importを書く場合、TypeScriptソースでも出力後の`.js`拡張子を指定します。

```ts
import { TaskItem } from './task-item.js';
```

ビルド後は、パッケージを一度tarballにして内容を確認します。

```sh
pnpm build
pnpm pack
```

生成された`.tgz`にJavaScript、Source Map、`.d.ts`が入り、不要なテストや設定ファイルが含まれていないかを見ます。

## Custom Elements Manifestで公開面を機械可読にする

Custom Elements Manifestは、要素、属性、プロパティ、イベント、Slot、CSS Partsなどを記述するJSON形式です。IDEやコンポーネントカタログがAPIを読み取れます。

```sh
pnpm add -D @custom-elements-manifest/analyzer
pnpm exec cem analyze
```

解析器が判断しにくい公開面はJSDocで補います。

```ts
/**
 * 1件のタスクを表示する。
 *
 * @tag task-item
 * @slot label - タスク名として表示する内容
 * @slot meta - 期限などの補助情報
 * @fires task-toggle - 完了状態の変更要求
 * @fires task-remove - 削除要求
 * @csspart container - タスク全体の外枠
 * @csspart remove-button - 削除ボタン
 * @cssprop [--task-item-accent-color=#2563eb] - 強調色
 */
export class TaskItem extends HTMLElement {}
```

生成された`custom-elements.json`もnpmパッケージに含めます。`package.json`の`customElements`フィールドから場所を示せます。

```json
{
  "customElements": "custom-elements.json"
}
```

## READMEには最小利用例と契約を書く

READMEの冒頭には、インストールから表示までの最短例を置きます。

```sh
pnpm add @example/task-elements
```

```ts
import '@example/task-elements/register';
```

```html
<task-item>
  <span slot="label">原稿をレビューする</span>
</task-item>
```

続けて、対象ブラウザ、属性とプロパティ、イベントdetail、Slot、Parts、CSS Custom Propertiesを記載します。内部のクラス名一覧は公開契約ではないため不要です。

## SemVerはHTMLとCSSの契約にも適用する

JavaScriptメソッドを削除する変更だけが破壊的変更ではありません。

- 属性名の変更
- イベント名または`detail`形式の変更
- Slot名の変更
- Part名の削除
- CSS Custom Propertyの意味変更
- 対象ブラウザの切り上げ

これらは利用者のHTML、CSS、型、テストを壊します。非推奨期間を置き、Major Versionで削除します。

パッケージ化によって、Custom Elementは単なるソースファイルから、名前と契約を持つ配布物になります。React、Vue、Angularとの接続方法は第21章で比較します。

## 参考資料

- [Getting Started - Custom Elements Manifest](https://custom-elements-manifest.open-wc.org/analyzer/getting-started/)
- [Node.js packages: package entry points](https://nodejs.org/api/packages.html#package-entry-points)
