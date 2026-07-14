---
title: "第34章 ネストしたRouteとレイアウト設計"
---

前章までで、URLとComponentを対応させ、遷移する方法を学びました。この章では、その対応をもう一段深めます。ページの中に、さらに切り替わる領域を持たせる「ネストしたRoute（子Route）」です。

たとえば、商品詳細ページに「基本情報」「レビュー」「関連商品」といったタブがあり、タブを切り替えると、詳細ページの一部だけが変わる、という画面を考えてみましょう。ページ全体は共通で、その内側だけがURLに応じて切り替わります。こうした入れ子の構造を、子Routeで表現します。あわせて、複数のページで共通するヘッダーやサイドバーを、レイアウトとしてまとめる設計も学びます。ネストは、規模の大きなアプリケーションを整理するうえで欠かせない考え方です。

:::message
**この章で学ぶこと**

- 子Route（ネストしたRoute）の定義
- ネストした`RouterOutlet`の仕組み
- 共通レイアウトの設計
- 既定の子Routeとリダイレクト
:::

## 子Routeを定義する

ネストは、ルート定義の`children`プロパティで表します。親のルートの中に、子のルートの配列を書きます。

```ts:src/app/app.routes.ts
export const routes: Routes = [
  {
    path: 'products/:id',
    component: ProductDetail,
    children: [
      { path: 'info', component: ProductInfo },
      { path: 'reviews', component: ProductReviews },
    ],
  },
];
```

この定義は、次のようなURLの階層を作ります。`/products/42/info`なら、`ProductDetail`の中に`ProductInfo`が表示されます。`/products/42/reviews`なら、同じ`ProductDetail`の中身が`ProductReviews`に切り替わります。親の`ProductDetail`は共通で、`children`で指定した部分だけがURLに応じて差し替わる、という構造です。

## ネストしたRouterOutlet

子Routeを表示するには、親のComponentのテンプレートに、もうひとつ`RouterOutlet`を置きます。第32章では、アプリの最上位に`<router-outlet />`を置きました。ネストでは、その内側にさらに`<router-outlet />`を置く、という入れ子になります。

```ts:src/app/product-detail.ts
import { Component, input } from '@angular/core';
import { RouterOutlet, RouterLink } from '@angular/router';

@Component({
  selector: 'app-product-detail',
  imports: [RouterOutlet, RouterLink],
  template: `
    <h1>商品 {{ id() }}</h1>
    <nav>
      <a routerLink="info">基本情報</a>
      <a routerLink="reviews">レビュー</a>
    </nav>
    <router-outlet />
  `,
})
export class ProductDetail {
  readonly id = input.required<string>();
}
```

`<h1>`とタブの`<nav>`は、どの子Routeでも共通して表示されます。そして、その下の`<router-outlet />`の位置に、`info`か`reviews`のどちらかが差し込まれます。最上位の`RouterOutlet`が`ProductDetail`を表示し、`ProductDetail`の中の`RouterOutlet`が、さらにその子を表示する。この二段構えが、ネストの仕組みです。

ここで、タブのリンクが`routerLink="info"`と、先頭にスラッシュのない相対パスになっている点に注目してください。スラッシュなしの相対パスは、現在のルートを基準に解釈されます。`/products/42`を見ているとき、`info`は`/products/42/info`を意味します。ネストの中では、この相対指定が扱いやすくなります。

## 共通レイアウトを設計する

ネストの考え方は、タブのような細かい単位だけでなく、アプリケーション全体のレイアウトにも使えます。多くのアプリには、複数のページで共通するヘッダーやサイドバーがあります。これを、レイアウト用のComponentとしてまとめ、その内側に各ページを差し込む、という設計が定番です。

```ts:src/app/app.routes.ts
export const routes: Routes = [
  {
    path: '',
    component: MainLayout, // ヘッダー・サイドバーを持つ枠
    children: [
      { path: 'home', component: Home },
      { path: 'products', component: ProductList },
      { path: 'settings', component: Settings },
    ],
  },
];
```

`MainLayout`は、ヘッダーやサイドバーと、中身を差し込む`<router-outlet />`だけを持つComponentです。

```ts:src/app/main-layout.ts
@Component({
  selector: 'app-main-layout',
  imports: [RouterOutlet, RouterLink],
  template: `
    <header>マイアプリ</header>
    <aside>
      <a routerLink="/home">ホーム</a>
      <a routerLink="/products">商品</a>
    </aside>
    <main>
      <router-outlet />
    </main>
  `,
})
export class MainLayout {}
```

こうすると、`/home`でも`/products`でも、ヘッダーとサイドバーは共通で表示され、`<main>`の中身だけがページごとに切り替わります。共通部分を一か所にまとめられるため、修正も一か所で済みます。ログイン画面など、レイアウトを持たせたくないページは、この`children`の外に定義すればよいので、レイアウトの適用範囲も柔軟に設計できます。

## 既定の子Routeとリダイレクト

ネストしたとき、親のURL（たとえば`/products/42`）にちょうど一致した場合、子の`RouterOutlet`には何を表示すべきでしょうか。何も指定がないと、`RouterOutlet`は空のままです。これを避けるため、既定の子Routeへリダイレクトする定義を加えるのが一般的です。

```ts:src/app/app.routes.ts
{
  path: 'products/:id',
  component: ProductDetail,
  children: [
    { path: '', redirectTo: 'info', pathMatch: 'full' }, // 既定はinfo
    { path: 'info', component: ProductInfo },
    { path: 'reviews', component: ProductReviews },
  ],
}
```

`{ path: '', redirectTo: 'info', pathMatch: 'full' }`により、`/products/42`にアクセスすると、自動的に`/products/42/info`へ転送されます。`pathMatch: 'full'`は、「URLが完全に空のときだけ転送する」という指定で、リダイレクトでは重要です。これがないと、意図しないパスまで転送してしまうことがあります。既定のタブや初期表示を決めたいときの、定番の書き方です。

## Routeごとにproviderを設定する

ネストしたRouteには、その範囲だけで有効なServiceを登録することもできます。ルート定義の`providers`プロパティを使います。

```ts:src/app/app.routes.ts
{
  path: 'admin',
  providers: [AdminService], // adminの範囲だけで使えるService
  children: [
    { path: 'users', component: AdminUsers },
    { path: 'teams', component: AdminTeams },
  ],
}
```

`admin`以下のページでだけ使う`AdminService`を、この範囲に限定して登録できます。第25章で学んだInjectorの階層が、ルートの単位でも働くわけです。管理画面など、特定の領域でだけ必要なServiceを、その範囲に閉じて提供したいときに役立ちます。アプリ全体で使うわけではないServiceを、`providedIn: 'root'`ではなくルート単位に置くことで、関心の範囲を明確にできます。

## よくあるつまずき

- **親に`RouterOutlet`を置き忘れる**: 子Routeを定義しても、親のテンプレートに`<router-outlet />`がなければ、子は表示されません。ネストの各段に`RouterOutlet`が要ります。
- **絶対パスと相対パスの混同**: ネスト内のリンクで先頭にスラッシュを付けると、ルートからの絶対パスになります。現在地を基準にしたいときは、スラッシュなしの相対パスを使います。
- **リダイレクトの`pathMatch`忘れ**: `redirectTo`を使うとき、`pathMatch: 'full'`を付け忘れると、意図しない範囲まで転送されることがあります。空パスからのリダイレクトでは、原則`'full'`を指定します。
- **ネストを深くしすぎる**: 子の中の子の、さらに子……と階層を深くすると、URLもコードも追いにくくなります。ネストは、画面の構造が実際に入れ子になっている場合に用い、無理に深い階層を作らないようにします。
- **レイアウトの適用範囲を誤る**: ログイン画面のように枠を持たせたくないページを、レイアウトの`children`に入れてしまうと、不要なヘッダーが表示されます。レイアウト外に置くべきページを、明確に切り分けます。

## まとめ

- ネストしたRouteは、ルート定義の`children`で表します
- 子Routeを表示するには、親のテンプレートに`<router-outlet />`を置きます
- 共通レイアウトは、ヘッダーなどと`RouterOutlet`を持つComponentを親に据えて実現します
- 既定の子Routeは、空パスから`redirectTo`（`pathMatch: 'full'`付き）で指定します
- ルート単位の`providers`で、その範囲だけに有効なServiceを登録できます

次章では、大きくなったアプリケーションを分割し、必要になったときに読み込むLazy Loadingを学びます。
