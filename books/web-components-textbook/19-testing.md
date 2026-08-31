---
title: "実ブラウザでのテスト"
---

Custom Elementsの登録とアップグレード、Shadow DOMの境界、フォーカスの移動、フォームへの参加、CSSの適用。ここまで本書で扱ってきた仕組みは、すべてブラウザが提供する機能です。だからテストも実ブラウザで動かします。DOMのエミュレーターは要素の生成や属性の読み書きまでは再現しますが、イベントがShadow DOMを越える経路、Form-associated Custom Elementの送信と検証、フォーカスリングやレイアウトの実際の振る舞いまでは一致しません。エンジンごとの実装差も、実ブラウザでなければ見えません。

本書ではPlaywrightを使い、Chromium、Firefox、WebKitの3エンジンで、利用者から見える振る舞いを検証します。

テストする対象は、内部のprivate methodではなく利用者が触る契約です。属性を変えた結果、画面とプロパティが一致するか。操作イベントがShadow DOMを越えて届くか。キーボードだけで操作できるか。この3つを実ブラウザで確かめておけば、内部DOMを作り替えてもテストはそのまま使えます。

## Playwrightを追加する

Viteプロジェクトへテストランナーを追加します。

```sh
pnpm add -D @playwright/test
pnpm exec playwright install
```

開発サーバーをテスト時に起動する設定を書きます。

```ts:playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  use: {
    baseURL: 'http://127.0.0.1:5173',
  },
  webServer: {
    command: 'pnpm dev --host 127.0.0.1',
    url: 'http://127.0.0.1:5173',
    reuseExistingServer: !process.env.CI,
  },
  projects: [
    { name: 'chromium', use: devices['Desktop Chrome'] },
    { name: 'firefox', use: devices['Desktop Firefox'] },
    { name: 'webkit', use: devices['Desktop Safari'] },
  ],
});
```

`package.json`へ実行コマンドを追加します。

```json
{
  "scripts": {
    "test:e2e": "playwright test"
  }
}
```

## テスト用のgalleryページを作る

各部品を単独で表示する小さなページがあると、手動確認と自動テストの入口を共有できます。

```html:gallery/task-item.html
<!doctype html>
<html lang="ja">
  <head>
    <meta charset="UTF-8">
  </head>
  <body>
    <task-item id="target">
      <span slot="label">原稿をレビューする</span>
    </task-item>
    <script type="module">
      import '/src/register.ts';

      document.querySelector('#target').task = {
        id: 'task-1',
        label: '原稿をレビューする',
        completed: false,
        priority: 'normal',
        createdAt: '2026-01-01T00:00:00.000Z',
      };
    </script>
  </body>
</html>
```

ひとつの状態を再現するページを用意すると、失敗をブラウザで直接開けます。

## roleと名前から操作する

PlaywrightのLocator（画面上の要素を特定するAPI）は、既定で`open`なShadow DOM内も探索します。内部CSSセレクターをたどらず、利用者が認識するroleと名前で操作できます。

```ts:tests/task-item.spec.ts
import { expect, test } from '@playwright/test';

test.beforeEach(async ({ page }) => {
  await page.goto('/gallery/task-item.html');
});

test('チェック操作を通知する', async ({ page }) => {
  const item = page.locator('task-item');
  const [detail] = await Promise.all([
    item.evaluate((element) => {
      return new Promise<unknown>((resolve) => {
        element.addEventListener('task-toggle', (received) => {
          resolve((received as CustomEvent).detail);
        }, { once: true });
      });
    }),
    page.getByRole('checkbox').check(),
  ]);

  expect(detail).toEqual({
    id: 'task-1',
    completed: true,
  });
});
```

gallery側で`task`プロパティを設定しておけば、イベントのdetailまで検証できます。

## 属性とプロパティの同期を確かめる

```ts
test('completedプロパティを属性へ反映する', async ({ page }) => {
  const item = page.locator('task-item');

  await item.evaluate((element) => {
    (element as HTMLElement & { completed: boolean }).completed = true;
  });

  await expect(item).toHaveAttribute('completed', '');
  await expect(page.getByRole('checkbox')).toBeChecked();
});
```

公開プロパティ、反映属性、利用者が見るチェック状態を同じテストで結びます。内部の`#render()`を直接呼ぶ必要はありません。

## フォーム送信と検証をブラウザで試す

```ts
test('task-inputの値がFormDataへ入る', async ({ page }) => {
  await page.goto('/gallery/task-form.html');

  await page.getByRole('textbox', { name: '新しいタスク' })
    .fill('テストを書く');
  await page.getByRole('button', { name: '追加' }).click();

  await expect(page.getByTestId('submitted-value'))
    .toHaveText('テストを書く');
});

test('必須入力が空なら送信しない', async ({ page }) => {
  await page.goto('/gallery/task-form.html');

  await page.getByRole('button', { name: '追加' }).click();

  await expect(page.getByRole('textbox')).toBeFocused();
  await expect(page.getByTestId('submitted-value')).toBeEmpty();
});
```

Form-associated Custom Elementの動作はブラウザ実装に関わるため、3エンジンでの実行に価値があります。

## キーボード操作をテストする

```ts
test('矢印キーでフィルターを移動できる', async ({ page }) => {
  await page.goto('/gallery/task-filter.html');

  const all = page.getByRole('button', { name: 'すべて' });
  const active = page.getByRole('button', { name: '未完了' });

  await all.focus();
  await page.keyboard.press('ArrowRight');

  await expect(active).toBeFocused();
});
```

クリックだけでなく、Tab、Enter、Space、矢印キーで主要操作を通します。自動テストに加え、フォーカスリングが見えるかは実画面でも確認します。

## closed Shadow Rootをテスト都合で選ばない

Playwrightの通常LocatorはopenなShadow DOMを探索しますが、closedなShadow Root内部は探索しません。closedを使う部品は、ホストの公開APIと外から観測できる結果だけでテストします。

テストから内部へ入りたいという理由だけで本番の`mode`を変えるのも、`closed`を選んだのに専用の裏口を作るのも避けます。`mode`は製品要件に基づいて選び、テストは公開契約に合わせます。

## 見た目の変更だけScreenshotで補う

色、余白、フォーカスリング、強制カラーモードはScreenshot比較が役立ちます。

```ts
await expect(page.locator('task-item'))
  .toHaveScreenshot('task-item-completed.png');
```

すべてを画像比較へ寄せると、文言やroleの意味を確かめにくくなります。振る舞いはLocatorとassertion、見た目だけScreenshotに分けます。

## まとめ

この章では、Custom Elementを実ブラウザでテストする方法を学びました。

- Custom Elements、Shadow DOM、フォーカス、フォームはブラウザの機能なので、実ブラウザで動かして確かめます。
- Playwrightの`webServer`設定で開発サーバーを起動し、Chromium、Firefox、WebKitの3エンジンで実行します。
- 部品を単独で表示するgalleryページを作ると、手動確認と自動テストが同じ入口を共有できます。
- Locatorはopen Shadow DOM内も探索するため、内部セレクターではなくroleと名前で操作します。
- プロパティ、反映属性、画面上の表示を1つのテストで結び付けます。
- フォーム送信と制約検証、キーボード操作は、実ブラウザだからこそ意味のある検証項目です。
- `mode`は製品要件で選び、テストの都合で変えません。
- 色や余白のような見た目だけScreenshot比較に任せ、振る舞いはassertionで書きます。

テストで契約を固定できたら、次はその契約を型やドキュメントと一緒に配布します。『npmパッケージ化とドキュメント』の章では、クラスと登録処理を分けてnpmパッケージにまとめる方法を扱います。

## 参考資料

- [Locators - Playwright](https://playwright.dev/docs/locators)
- [Playwright Test](https://playwright.dev/docs/intro)
