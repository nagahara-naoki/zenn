---
title: "Lazy LoadingとFeature分割"
---

アプリケーションが大きくなると、画面の数も、それを構成するコードの量も増えていきます。何も対策しないと、利用者が最初にページを開いたとき、アプリのすべてのコードをまとめてダウンロードすることになります。ほとんど使わない管理画面のコードまで、最初に読み込むのは無駄です。

この問題を解決するのが、Lazy Loading（遅延読み込み）です。コードを画面ごとの塊に分け、その画面へ実際に移動したときに、はじめて対応する塊を読み込みます。この章では、Lazy Loadingの書き方と、それを前提にしたFeature（機能）単位のアプリケーション分割を学びます。パフォーマンスと保守性の両面で効く、実務に直結するテーマです。

:::message
**この章で学ぶこと**

- Lazy Loadingが解決する課題
- `loadComponent`による単一Componentの遅延読み込み
- `loadChildren`によるRoute群の遅延読み込み
- 先読み（Preloading）という調整
:::

## Lazy Loadingが解決する課題

まず、Lazy Loadingがない場合を考えます。この場合、アプリのすべてのComponentが、ひとつの大きなJavaScriptの塊（バンドル）にまとめられます。利用者がトップページを開くと、この塊がまるごとダウンロードされます。画面が表示されるまでの時間は、この塊の大きさに比例します。アプリが大きくなるほど、最初の表示が遅くなるのです。

しかし、利用者が最初に必要とするのは、最初に見る画面のコードだけです。まだ開いていないページや、権限がなくて開けない管理画面のコードは、その時点では要りません。Lazy Loadingは、この「いま要るものだけ読み込む」を実現します。コードをページ単位の塊に分割し、それぞれのページへ移ったときに、対応する塊を追加で読み込むのです。

結果として、最初にダウンロードするコードの量が減り、初回表示が速くなります。第7章で、Standalone Componentが遅延読み込みをしやすくすると述べましたが、その効果がここで具体化します。NgModule時代にも遅延読み込みはありましたが、モジュール単位でしか分割できませんでした。Standaloneでは、Component1つからでも遅延読み込みでき、より細かく、柔軟に分割できるようになっています。

## loadComponentで単一Componentを遅延読み込み

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

## loadChildrenでRoute群を遅延読み込み

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

## Feature単位で分割する

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

## 先読みで遷移を速くする

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

## 分割されたことを確かめる

Lazy Loadingが効いているかは、本番ビルドの出力で確認できます。第3章で学んだ`ng build`を実行すると、生成されたJavaScriptの一覧が表示されます。遅延読み込みが設定されていれば、最初に読み込む塊（メインバンドル）とは別に、遅延読み込み対象ごとの塊（チャンク）が分かれて出力されます。

```bash
ng build
```

出力される一覧に、メインの塊と、機能ごとに分かれた小さな塊が並んでいれば、分割が効いている証拠です。逆に、すべてがひとつの巨大な塊にまとまっているなら、遅延読み込みが設定できていないか、意図せず全体が結合されている可能性があります。ビルド結果を確認する習慣をつけると、分割の効果を目で確かめられます。

## GuardとLazy Loadingの組み合わせ

Lazy Loadingは、次章で学ぶGuardと組み合わせると、より効果的になります。とくに`CanMatch`というGuardは、遅延読み込みの前に「そのルートを使うかどうか」を判断できます。

たとえば、管理者だけがアクセスできる`admin`機能を考えます。`CanMatch`で管理者かどうかを判定すれば、権限のない利用者に対しては、`admin`のコードそのものをダウンロードさせずに済みます。アクセス制御と、コードの読み込みの節約を、同時に実現できるわけです。Guardの詳細は次章で扱いますが、Lazy Loadingと権限管理が結びつく点は、覚えておく価値があります。

## よくあるつまずき

- **`component`と`loadComponent`の併用**: ひとつのルートに、`component`と`loadComponent`の両方は書けません。遅延読み込みするなら`loadComponent`だけを使います。
- **`import()`のパス誤り**: 動的インポートのパスが間違っていると、そのページへ移動したときに初めてエラーになります。読み込み対象のパスとエクスポート名を確認します。
- **分割しすぎ**: あまりに細かく分割すると、塊の数が増え、かえって遷移のたびの読み込みが煩雑になります。機能のまとまりを単位に、適度な粒度で分けます。
- **共有コードの重複**: 複数の遅延読み込み対象が同じServiceやComponentを使う場合、それらが各塊に重複して含まれることがあります。ビルドツールがある程度まとめてくれますが、共有が多いものは、遅延させずに共通の場所へ置くほうがよい場合もあります。

## まとめ

- Lazy Loadingは、コードをページ単位の塊に分け、必要になったときに読み込む仕組みです
- 単一Componentは`loadComponent`、Route群は`loadChildren`で遅延読み込みします
- いずれも動的インポート（`import()`）を使い、対象のコードを遅延させます
- `loadChildren`は、アプリを機能（Feature）単位で分割する設計につながります
- 先読み（`withPreloading`）で、遅延読み込みと遷移速度を両立できます

次章では、ルートへのアクセスを制御するGuardと、遷移前にデータを用意するResolverを、関数型を軸に学びます。
