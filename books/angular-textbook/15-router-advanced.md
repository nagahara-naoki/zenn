---
title: "ルーティング応用（ネスト・Lazy Loading・Guard）"
---

この章では、Routerの応用的な機能を学びます。ネストしたRouteとレイアウト、必要になったときに読み込むLazy Loading、そしてアクセスを制御するGuardとResolverを扱います。

:::message
**この章で学ぶこと**

- 子Route（ネストしたRoute）の定義
- ネストした`RouterOutlet`の仕組み
- Lazy Loadingが解決する課題
- `loadComponent`による単一Componentの遅延読み込み
- Route Guardによるアクセス制御
- 関数型Guardの書き方と種類
:::

## ネストしたRouteとレイアウト設計

前章までで、URLとComponentを対応させ、遷移する方法を学びました。この節では、その対応をもう一段深めます。ページの中に、さらに切り替わる領域を持たせる「ネストしたRoute（子Route）」です。

たとえば、商品詳細ページに「基本情報」「レビュー」「関連商品」といったタブがあり、タブを切り替えると、詳細ページの一部だけが変わる、という画面を考えてみましょう。ページ全体は共通で、その内側だけがURLに応じて切り替わります。こうした入れ子の構造を、子Routeで表現します。あわせて、複数のページで共通するヘッダーやサイドバーを、レイアウトとしてまとめる設計も学びます。ネストは、規模の大きなアプリケーションを整理するうえで欠かせない考え方です。

### 子Routeを定義する

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

### ネストしたRouterOutlet

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

### 共通レイアウトを設計する

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

### 既定の子Routeとリダイレクト

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

### Routeごとにproviderを設定する

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

### よくあるつまずき

- **親に`RouterOutlet`を置き忘れる**: 子Routeを定義しても、親のテンプレートに`<router-outlet />`がなければ、子は表示されません。ネストの各段に`RouterOutlet`が要ります。
- **絶対パスと相対パスの混同**: ネスト内のリンクで先頭にスラッシュを付けると、ルートからの絶対パスになります。現在地を基準にしたいときは、スラッシュなしの相対パスを使います。
- **リダイレクトの`pathMatch`忘れ**: `redirectTo`を使うとき、`pathMatch: 'full'`を付け忘れると、意図しない範囲まで転送されることがあります。空パスからのリダイレクトでは、原則`'full'`を指定します。
- **ネストを深くしすぎる**: 子の中の子の、さらに子……と階層を深くすると、URLもコードも追いにくくなります。ネストは、画面の構造が実際に入れ子になっている場合に用い、無理に深い階層を作らないようにします。
- **レイアウトの適用範囲を誤る**: ログイン画面のように枠を持たせたくないページを、レイアウトの`children`に入れてしまうと、不要なヘッダーが表示されます。レイアウト外に置くべきページを、明確に切り分けます。

## Lazy LoadingとFeature分割

アプリケーションが大きくなると、画面の数も、それを構成するコードの量も増えていきます。何も対策しないと、利用者が最初にページを開いたとき、アプリのすべてのコードをまとめてダウンロードすることになります。ほとんど使わない管理画面のコードまで、最初に読み込むのは無駄です。

この問題を解決するのが、Lazy Loading（遅延読み込み）です。コードを画面ごとの塊に分け、その画面へ実際に移動したときに、はじめて対応する塊を読み込みます。この節では、Lazy Loadingの書き方と、それを前提にしたFeature（機能）単位のアプリケーション分割を学びます。パフォーマンスと保守性の両面で効く、実務に直結するテーマです。

### Lazy Loadingが解決する課題

まず、Lazy Loadingがない場合を考えます。この場合、アプリのすべてのComponentが、ひとつの大きなJavaScriptの塊（バンドル）にまとめられます。利用者がトップページを開くと、この塊がまるごとダウンロードされます。画面が表示されるまでの時間は、この塊の大きさに比例します。アプリが大きくなるほど、最初の表示が遅くなるのです。

しかし、利用者が最初に必要とするのは、最初に見る画面のコードだけです。まだ開いていないページや、権限がなくて開けない管理画面のコードは、その時点では要りません。Lazy Loadingは、この「いま要るものだけ読み込む」を実現します。コードをページ単位の塊に分割し、それぞれのページへ移ったときに、対応する塊を追加で読み込むのです。

結果として、最初にダウンロードするコードの量が減り、初回表示が速くなります。第7章で、Standalone Componentが遅延読み込みをしやすくすると述べましたが、その効果がここで具体化します。NgModule時代にも遅延読み込みはありましたが、モジュール単位でしか分割できませんでした。Standaloneでは、Component1つからでも遅延読み込みでき、より細かく、柔軟に分割できるようになっています。

### loadComponentで単一Componentを遅延読み込み

もっとも基本的なのが、`loadComponent`です。ルート定義で`component`の代わりに`loadComponent`を使い、動的インポート（`import()`）で対象のComponentを指定します。

```ts:src/app/app.routes.ts
export const routes: Routes = [
  { path: 'home', component: Home },
  {
    path: 'settings',
    loadComponent: () => import('./settings/settings').then((m) => m.Settings),
  },
];
```

`loadComponent`には、Componentを返す関数を渡します。この関数の中で`import()`を使うのが肝心です。`import()`は、その行が実行されるまでファイルを読み込まない、動的な取り込みです。つまり、利用者が`/settings`へ移動して、はじめて`settings`のコードがダウンロードされます。`/home`しか見ない利用者は、`settings`のコードを一切読み込みません。

なお、対象のComponentがそのファイルの既定エクスポート（`export default`）であれば、`.then((m) => m.Settings)`を省いて、より短く書けます。単一の画面を遅延読み込みするなら、この`loadComponent`が第一の選択肢です。

### loadChildrenでRoute群を遅延読み込み

画面が1つではなく、関連する複数の画面をまとめて遅延読み込みしたいこともあります。管理画面のように、その下にさらに複数のページを持つ場合です。このときは、`loadChildren`を使い、Routes（ルート定義の配列）を別ファイルから読み込みます。

```ts:src/app/app.routes.ts
export const routes: Routes = [
  { path: 'home', component: Home },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes').then((m) => m.adminRoutes),
  },
];
```

読み込まれる側の`admin.routes.ts`には、その機能に属するルートをまとめて定義します。

```ts:src/app/admin/admin.routes.ts
import { Routes } from '@angular/router';

export const adminRoutes: Routes = [
  { path: '', component: AdminDashboard },
  { path: 'users', component: AdminUsers },
  { path: 'teams', component: AdminTeams },
];
```

こうすると、`/admin`以下のページ群が、ひとつの塊として遅延読み込みされます。利用者が`/admin`に足を踏み入れたときに、`admin`関連のコードがまとめて読み込まれ、それ以外の利用者はまったく読み込みません。機能のまとまりごとに、コードを分割できるわけです。

### Feature単位で分割する

`loadChildren`は、単なるパフォーマンスの工夫にとどまりません。アプリケーションを機能（Feature）の単位で分割し、整理する設計につながります。

たとえば、商品機能、注文機能、管理機能といった、業務上のまとまりごとに、フォルダとルートを分けます。

```text
src/app/
├── product/
│   ├── product.routes.ts
│   └── ...
├── order/
│   ├── order.routes.ts
│   └── ...
└── admin/
    ├── admin.routes.ts
    └── ...
```

そして、最上位のルートから、各機能の`routes`を`loadChildren`で読み込みます。こうすると、機能ごとにコードが独立し、担当を分けやすく、影響範囲も限定されます。パフォーマンスの分割単位と、設計上の分割単位が一致するのです。第11部で扱うアーキテクチャ設計でも、このFeature単位の分割は重要な柱になります。ひとつの機能が肥大化しても、ほかの機能のコードには影響が及ばない、という独立性が保てます。

### 先読みで遷移を速くする

Lazy Loadingには、ひとつ引き換えとなる点があります。遅延読み込みするページへ初めて移動するとき、その塊をダウンロードするため、わずかな待ちが生じます。初回表示は速くなる代わりに、各ページの初回遷移で少し待つ、というわけです。

この待ちをやわらげるのが、先読み（Preloading）です。最初の画面を表示し終えて手が空いたタイミングで、遅延読み込みするコードを裏で先に読み込んでおく仕組みです。第32章で触れた`provideRouter()`の`with〜`機能で設定します。

```ts:src/app/app.config.ts
import { provideRouter, withPreloading, PreloadAllModules } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withPreloading(PreloadAllModules)),
  ],
};
```

`PreloadAllModules`は、遅延読み込み対象をすべて先読みする戦略です。初回表示は遅延読み込みの恩恵で速く保ちつつ、その後の遷移では、先読み済みのため待ちが減ります。両方のよいところを取れるわけです。より細かく、特定のページだけを先読みする独自の戦略を作ることもできます。アプリの規模や利用のされ方に応じて、調整できます。

### 分割されたことを確かめる

Lazy Loadingが効いているかは、本番ビルドの出力で確認できます。第3章で学んだ`ng build`を実行すると、生成されたJavaScriptの一覧が表示されます。遅延読み込みが設定されていれば、最初に読み込む塊（メインバンドル）とは別に、遅延読み込み対象ごとの塊（チャンク）が分かれて出力されます。

```bash
ng build
```

出力される一覧に、メインの塊と、機能ごとに分かれた小さな塊が並んでいれば、分割が効いている証拠です。逆に、すべてがひとつの巨大な塊にまとまっているなら、遅延読み込みが設定できていないか、意図せず全体が結合されている可能性があります。ビルド結果を確認する習慣をつけると、分割の効果を目で確かめられます。

### GuardとLazy Loadingの組み合わせ

Lazy Loadingは、次章で学ぶGuardと組み合わせると、より効果的になります。とくに`CanMatch`というGuardは、遅延読み込みの前に「そのルートを使うかどうか」を判断できます。

たとえば、管理者だけがアクセスできる`admin`機能を考えます。`CanMatch`で管理者かどうかを判定すれば、権限のない利用者に対しては、`admin`のコードそのものをダウンロードさせずに済みます。アクセス制御と、コードの読み込みの節約を、同時に実現できるわけです。Guardの詳細は次章で扱いますが、Lazy Loadingと権限管理が結びつく点は、覚えておく価値があります。

### よくあるつまずき

- **`component`と`loadComponent`の併用**: ひとつのルートに、`component`と`loadComponent`の両方は書けません。遅延読み込みするなら`loadComponent`だけを使います。
- **`import()`のパス誤り**: 動的インポートのパスが間違っていると、そのページへ移動したときに初めてエラーになります。読み込み対象のパスとエクスポート名を確認します。
- **分割しすぎ**: あまりに細かく分割すると、塊の数が増え、かえって遷移のたびの読み込みが煩雑になります。機能のまとまりを単位に、適度な粒度で分けます。
- **共有コードの重複**: 複数の遅延読み込み対象が同じServiceやComponentを使う場合、それらが各塊に重複して含まれることがあります。ビルドツールがある程度まとめてくれますが、共有が多いものは、遅延させずに共通の場所へ置くほうがよい場合もあります。

## Route GuardとResolverの新旧比較

第7部の締めくくりとして、ルーティングに関門を設ける2つの仕組み、GuardとResolverを学びます。Guardは「このページに入ってよいか」を判断する門番、Resolverは「このページを表示する前にデータをそろえる」下ごしらえ役です。どちらも、遷移のタイミングに割り込んで処理を行います。

これらの書き方も、モダンAngularで大きく変わりました。かつてはクラスとインターフェースで実装していましたが、現在は関数で書きます。この関数型のGuard・Resolverは、第24章で学んだ`inject()`があってこそ成り立つものです。この節では、関数型を主に、旧来のクラス型と比較しながら、アクセス制御と事前データ取得を学びます。

### Route Guardとは

Route Guardは、ルートへの遷移を許可するか、拒否するかを判断する仕組みです。もっとも多い用途は、認証によるアクセス制御です。「ログインしていない利用者は、マイページに入れない」といった制御を、Guardで実現します。

Guardは、遷移が起きようとするタイミングで呼ばれます。Guardが`true`を返せば遷移が進み、`false`を返せば遷移が止まります。あるいは、`UrlTree`（別のURLへの指示）を返して、ログインページへ振り向けることもできます。門番が通行の可否を判断し、必要なら別の場所へ案内する、というイメージです。

### 関数型Guardを書く

現在のGuardは、関数として書きます。もっとも代表的な、アクセス可否を判断する`CanActivateFn`を見てみましょう。

```ts:src/app/auth-guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth';

export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isLoggedIn()) {
    return true;
  }
  // 未ログインならログインページへ振り向ける
  return router.createUrlTree(['/login']);
};
```

`authGuard`は、ただの関数です。中で`inject()`を使い、`AuthService`と`Router`を受け取っています。ログイン済みなら`true`を返して遷移を許可し、そうでなければ`createUrlTree`でログインページへの指示を返します。第24章で「関数型のGuardは`inject()`があって初めて成り立つ」と述べたのは、このことです。関数の中で依存を注入できるからこそ、この簡潔な書き方ができます。

作ったGuardは、ルート定義の`canActivate`に指定します。

```ts:src/app/app.routes.ts
export const routes: Routes = [
  { path: 'mypage', component: MyPage, canActivate: [authGuard] },
];
```

`canActivate: [authGuard]`とすると、`/mypage`へ遷移しようとするたびに`authGuard`が呼ばれ、その戻り値で遷移の可否が決まります。配列なので、複数のGuardを並べることもでき、その場合はすべてが許可したときだけ遷移が進みます。

### Guardの種類

Guardには、判断するタイミングや対象によって、いくつかの種類があります。いずれも関数型で書けます。

| Guard | 役割 |
|---|---|
| `CanActivateFn` | そのルートに入ってよいかを判断する |
| `CanActivateChildFn` | 子ルートすべてへの進入を判断する |
| `CanDeactivateFn` | そのルートから離れてよいかを判断する |
| `CanMatchFn` | そのルート定義自体を使うかを判断する |

とくに`CanDeactivateFn`は、実務でよく使います。「入力途中のフォームがあるのに、ページを離れようとしたら確認する」という制御です。

```ts:src/app/unsaved-changes-guard.ts
import { CanDeactivateFn } from '@angular/router';

export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) => {
  if (component.hasUnsavedChanges()) {
    return confirm('保存していない変更があります。移動しますか');
  }
  return true;
};
```

`CanMatchFn`も特徴的です。これは、ルート定義そのものを「使うかどうか」を判断します。拒否すると、そのルートは存在しないものとして扱われ、ほかの候補が探されます。機能フラグによる出し分けや、権限に応じて別のComponentを見せる、といった高度な制御に使えます。

### Resolverで事前にデータを用意する

Resolverは、ページを表示する前に、必要なデータを取得しておく仕組みです。「商品詳細ページを表示する前に、その商品のデータを取得しておく」といった下ごしらえを担います。関数型では`ResolveFn`を使います。

```ts:src/app/product-resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { ProductService } from './product';

export const productResolver: ResolveFn<Product> = (route) => {
  const service = inject(ProductService);
  const id = route.paramMap.get('id')!;
  return service.findById(id);
};
```

ルート定義の`resolve`に指定すると、遷移が完了する前にこの関数が実行され、取得したデータがルートに紐づきます。

```ts:src/app/app.routes.ts
{
  path: 'products/:id',
  component: ProductDetail,
  resolve: { product: productResolver },
}
```

Component側では、解決されたデータを受け取ります。第32章で設定した`withComponentInputBinding()`があれば、解決データ（ここでは`product`）も`input()`で受け取れます。Resolverを使うと、「データが来るまでのローディング状態」をComponent側で扱わずに済み、表示された時点でデータがそろっている、という作りにできます。

ただし、Resolverには注意点もあります。データの取得が終わるまで遷移が完了しないため、取得が遅いと、遷移そのものが待たされます。時間のかかるデータは、Resolverで待たせるより、ページを先に表示してから読み込むほうが、体感がよい場合もあります。何をResolverで先に用意し、何をあとから読み込むかは、設計上の判断になります。

### 旧来のクラス型との比較

これらのGuardやResolverは、かつてはクラスとインターフェースで実装していました。同じ`authGuard`を、旧来の書き方で示します。

```ts:旧来の書き方（クラス型Guard）
import { Injectable, inject } from '@angular/core';
import { CanActivate, Router } from '@angular/router';

@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  private readonly auth = inject(AuthService);
  private readonly router = inject(Router);

  canActivate(): boolean | UrlTree {
    return this.auth.isLoggedIn() ? true : this.router.createUrlTree(['/login']);
  }
}
```

`CanActivate`インターフェースを実装したServiceとして書き、`canActivate`メソッドに判断を書いていました。ルート定義にはクラスを指定します。動作は関数型と同じですが、Serviceのクラスを1つ用意する必要があり、記述量が多くなります。

関数型は、これを大幅に簡潔にします。クラスもインターフェースも不要で、関数を1つエクスポートするだけです。`inject()`で依存を受け取れるため、Serviceにする必要もありません。テストのときも、関数を呼ぶだけで確かめられます。こうした利点から、モダンAngularでは関数型が標準となり、クラス型のGuardインターフェースは非推奨とされています。

:::message
既存のクラス型Guard・Resolverは、現在も動作します。ただし新規に書くものは関数型を使い、既存分は移行を検討するのがよいでしょう。関数型は`inject()`と組み合わさることで、Angular全体の関数中心の書き方に自然になじみます。
:::

### よくあるつまずき

- **Guardの中で`inject()`を関数の外に置く**: 関数型Guardの`inject()`は、関数本体の中で呼びます。Guardが呼ばれる文脈は注入コンテキストとして扱われるため、その中で依存を受け取れます。
- **`false`とリダイレクトを混同する**: 単に`false`を返すと、遷移が止まるだけで、その場にとどまります。ログインページなどへ誘導したいなら、`UrlTree`を返します。「拒否」と「別の場所へ案内」は別物です。
- **Resolverで重いデータを待たせる**: 取得の遅いデータをResolverに入れると、遷移全体が待たされ、操作が固まったように見えます。時間のかかるものは、ページ表示後に読み込むほうが体感がよい場合があります。
- **Guardの副作用に頼る**: Guardは可否の判断に徹します。Guardの中でデータを書き換えるなどの副作用を持たせると、遷移が中断されたときに整合性が崩れます。

## まとめ

- ネストしたRouteは、ルート定義の`children`で表します
- 子Routeを表示するには、親のテンプレートに`<router-outlet />`を置きます
- 共通レイアウトは、ヘッダーなどと`RouterOutlet`を持つComponentを親に据えて実現します
- Lazy Loadingは、コードをページ単位の塊に分け、必要になったときに読み込む仕組みです
- 単一Componentは`loadComponent`、Route群は`loadChildren`で遅延読み込みします
- いずれも動的インポート（`import()`）を使い、対象のコードを遅延させます
- Guardは、ルートへの遷移の可否を判断する門番の仕組みです
- 関数型Guardは`inject()`で依存を受け取り、`canActivate`などに指定します
- Guardには`CanActivateFn`・`CanDeactivateFn`・`CanMatchFn`などの種類があります

次章からは、非同期処理を扱う強力なライブラリRxJSに進みます。
