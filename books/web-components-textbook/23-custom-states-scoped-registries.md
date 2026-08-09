---
title: "新しいAPIは機能検出し、段階的に採用する"
---

Web Componentsは現在も拡張されています。Custom Statesは内部状態をCSSへ安全に公開し、Scoped Custom Element Registriesは要素名の定義範囲をShadow Rootへ閉じます。

Custom Statesは主要ブラウザへ広がっています。Scoped Registryは2026年時点でも一部の広く使われるブラウザで未対応です。同じ「新しいAPI」としてまとめず、互換性に応じて採用段階を変えます。

この章は二部構成です。前半では、安定した強化としてCustom Statesを扱います。後半では、利用できる環境がまだ限られるScoped Custom Element Registriesを扱います。

## Custom Statesで状態名だけをCSSへ渡す

内部状態をスタイルへ反映するためだけに属性を増やすと、HTML上の公開面が広がります。`ElementInternals.states`へ状態名を追加すると、`:state()`疑似クラスから選べます。

```ts
class TaskFilter extends HTMLElement {
  #internals = this.attachInternals();
  #filter: 'all' | 'active' | 'completed' = 'all';

  get filter(): 'all' | 'active' | 'completed' {
    return this.#filter;
  }

  set filter(value: 'all' | 'active' | 'completed') {
    this.#filter = value;

    for (const name of ['all', 'active', 'completed']) {
      if (name === value) {
        this.#internals.states.add(name);
      } else {
        this.#internals.states.delete(name);
      }
    }
  }
}
```

ページ側とShadow Root内部の両方から選択できます。

```css
task-filter:state(active) {
  border-color: #2563eb;
}
```

```css
:host(:state(active)) button[data-filter='active'] {
  color: white;
  background: #2563eb;
}
```

## 属性とCustom Stateのどちらを使うか

HTMLから設定でき、URL保存やCSSだけの利用でも意味がある状態は属性に向きます。`disabled`、`required`、`completed`などです。

読み込み中、内部アニメーションの段階、計算途中の状態など、利用者が直接設定すべきでない値はCustom Statesに向きます。

Custom Statesはprivate fieldではありません。CSSから状態名を観測できる公開契約です。名前を変更すると利用者のCSSが壊れるため、ManifestとREADMEへ記載します。

## Scoped Registryは名前をShadow Rootへ閉じる

通常の`window.customElements`は文書全体で共有されます。同じページへ2つのライブラリが`<ui-button>`を登録しようとすると、後から登録した側が失敗します。

Scoped Custom Element Registryは、特定のShadow Rootだけで使うRegistryを作ります。

```ts
const registry = new CustomElementRegistry();

registry.define('ui-button', class extends HTMLElement {
  connectedCallback(): void {
    this.textContent = 'このShadow Root専用のボタン';
  }
});

const host = document.createElement('section');
const root = host.attachShadow({
  mode: 'open',
  customElementRegistry: registry,
});

root.innerHTML = '<ui-button></ui-button>';
document.body.append(host);
```

別のShadow Rootでは、同じ`ui-button`名へ別のクラスを登録できます。複数バージョンの共存、テストごとの定義、内部依存の名前衝突を解決できます。

## 既存のShadow Rootへ後から関連付ける

Registryを後から用意する場合、Shadow Rootを`null`のRegistryで作り、`initialize()`で関連付けられます。

```ts
const root = host.attachShadow({
  mode: 'open',
  customElementRegistry: null,
});

root.innerHTML = '<ui-button></ui-button>';
registry.initialize(root);
```

Declarative Shadow DOMでは、`shadowrootcustomelementregistry`属性を付けて後から関連付ける意思を示します。

```html
<component-host>
  <template
    shadowrootmode="open"
    shadowrootcustomelementregistry
  >
    <ui-button></ui-button>
  </template>
</component-host>
```

## 対応ブラウザが限られる機能を基本設計にしない

Scoped Registryを主要機能の前提にすると、未対応ブラウザへ大きなpolyfillや別bundleが必要になります。まずグローバルRegistryでも衝突しにくい接頭辞を付けます。Scoped Registryは、対応ブラウザでShadow Root内の依存要素だけに適用する追加機能として導入します。

```ts
function supportsScopedRegistry(): boolean {
  try {
    const registry = new CustomElementRegistry();
    const host = document.createElement('div');
    const root = host.attachShadow({
      mode: 'open',
      customElementRegistry: registry,
    });
    return root.customElementRegistry === registry;
  } catch {
    return false;
  }
}
```

TypeScriptの`lib.dom.d.ts`が新しいoptionへ追いついていない場合は、局所的な型定義を追加します。広い`any`でアプリケーション全体の型を弱めないようにします。

## 新機能を採用する4段階

1. HTML Living StandardとMDNで仕様と互換性を確認する
2. Chromium、Firefox、WebKitの対象バージョンで小さな検証を書く
3. 機能検出とフォールバックを実装する
4. Playwrightの3ブラウザprojectで継続テストする

新しいAPIを知っていることと、本番の基本経路へ置けることは別です。互換性がそろうまでは、既存APIで動く中核を残します。

手作業の更新処理が大きくなった場合は、第24章のLitで定型処理を減らせます。公開契約はそのまま維持します。

## 参考資料

- [CustomStateSet - MDN](https://developer.mozilla.org/en-US/docs/Web/API/CustomStateSet)
- [Using custom elements: Scoped registries - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements#scoped_custom_element_registries)
- [ShadowRoot.customElementRegistry - MDN](https://developer.mozilla.org/en-US/docs/Web/API/ShadowRoot/customElementRegistry)
