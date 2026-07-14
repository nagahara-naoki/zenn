---
title: "第32章 RouterModuleからprovideRouter()へ"
---

前章で、ルーティングがRoutes・`provideRouter()`・`RouterOutlet`・`routerLink`で構成されることを見ました。この章では、その設定を実際に書いていきます。中心となるのが、Routesの定義と、それをアプリケーションに登録する`provideRouter()`です。

ルーティングの設定方法は、Angularの歴史の中で変わってきました。かつては`RouterModule`というNgModuleを使っていましたが、現在は`provideRouter()`という関数を使います。これは、第7章で学んだ「モジュールから関数へ」という流れの、ルーティング版です。この章では、現在の`provideRouter()`を主に、旧来の`RouterModule`と比較しながら、ルーティングの基本設定を身につけます。

:::message
**この章で学ぶこと**

- Routesによるルート定義
- `provideRouter()`によるルーティングの登録
- `RouterOutlet`によるComponentの表示
- 旧来の`RouterModule`との違い
:::

## Routesを定義する

ルーティングの出発点は、Routesの定義です。「どのURLのときに、どのComponentを表示するか」を、配列で書きます。

```ts:src/app/app.routes.ts
import { Routes } from '@angular/router';
import { Home } from './home';
import { About } from './about';
import { NotFound } from './not-found';

export const routes: Routes = [
  { path: '', component: Home },
  { path: 'about', component: About },
  { path: '**', component: NotFound },
];
```

各要素は、`path`（URLの一部）と`component`（表示するComponent）の組です。`path: ''`は、何も付かないトップのURLを表します。`path: 'about'`は`/about`に対応します。最後の`path: '**'`は、どのルートにも一致しなかったときの受け皿で、404ページの表示によく使います。この定義が、URLと画面の対応表になります。ルート定義は、アプリケーションのページ構成をそのまま映す設計図でもあり、これを見れば、どんな画面が存在するのかを一望できます。

ルート定義には、`path`と`component`以外にも、いくつかのプロパティが使えます。よく使うものを挙げます。

- **`redirectTo`**: 別のパスへ転送します。`{ path: '', redirectTo: 'home', pathMatch: 'full' }`のように使います。
- **`pathMatch`**: パスの一致のしかたを指定します。`'full'`はURL全体の一致、`'prefix'`（既定）は前方一致です。
- **`title`**: そのページのタイトル（ブラウザのタブに表示される文字列）を指定します。
- **`children`**: 子のルートを定義します。第34章で扱います。

## provideRouterで登録する

定義したRoutesを、アプリケーションに登録します。第4章で見た`app.config.ts`の`providers`に、`provideRouter(routes)`を加えます。

```ts:src/app/app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
  ],
};
```

`provideRouter(routes)`だけでも動きますが、ここでは`withComponentInputBinding()`という機能を一緒に渡しています。これは、URLのパラメーターを、Componentの`input()`へ自動で結びつける機能です。第33章で活用するので、いまは「あとで役立つ設定を加えた」と捉えてください。`provideRouter()`は、こうした機能を`with〜`という関数で、必要な分だけ追加できる作りになっています。

主な`with〜`機能には、次のようなものがあります。

- **`withComponentInputBinding()`**: ルートパラメーターをComponentの入力に結びつけます。
- **`withHashLocation()`**: URLを`/#/about`のようなハッシュ形式にします。
- **`withInMemoryScrolling()`**: 画面遷移時のスクロール位置を制御します。
- **`withPreloading()`**: 遅延読み込みするコードを、先読みします（第35章で扱います）。

## RouterOutletで表示する

Routesを登録しただけでは、まだ画面には何も現れません。表示するComponentを差し込む場所を、テンプレートに用意する必要があります。それが`RouterOutlet`です。ルートのComponent（多くは`App`）のテンプレートに、`<router-outlet />`を置きます。

```ts:src/app/app.ts
import { Component } from '@angular/core';
import { RouterOutlet, RouterLink } from '@angular/router';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet, RouterLink],
  template: `
    <nav>
      <a routerLink="/">ホーム</a>
      <a routerLink="/about">概要</a>
    </nav>
    <router-outlet />
  `,
})
export class App {}
```

`<router-outlet />`の位置に、現在のURLに対応するComponentが表示されます。URLが`/about`なら`About`が、`/`なら`Home`が、この位置に描かれます。ヘッダーの`<nav>`は`RouterOutlet`の外にあるので、ページを移っても残り続けます。これが、SPAで「共通部分は保ちつつ、一部だけ差し替える」という動きの正体です。

なお、`RouterOutlet`や`routerLink`も、Standaloneの部品なので、使うComponentの`imports`に宣言します。第6章で学んだ`imports`の考え方が、ここでも通じます。

## 旧来のRouterModuleとの比較

`provideRouter()`が登場する前は、`RouterModule`というNgModuleでルーティングを設定していました。同じRoutesを、旧来の書き方で登録すると、次のようになります。

```ts:旧来の書き方（RouterModule）
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

const routes: Routes = [
  { path: '', component: Home },
  { path: 'about', component: About },
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
})
export class AppRoutingModule {}
```

ルートのアプリでは`RouterModule.forRoot(routes)`を、機能ごとのモジュールでは`RouterModule.forChild(routes)`を使い分ける必要がありました。専用のモジュール（`AppRoutingModule`）を作り、それをアプリのモジュールにimportする、という手順も要りました。

現在の`provideRouter()`は、これをぐっと簡潔にします。専用のモジュールは不要で、`app.config.ts`に一行加えるだけです。`forRoot`と`forChild`の区別もありません。第7章で述べた「モジュールをimportする書き方から、`provide〜`関数を並べる書き方へ」という流れが、ルーティングにもはっきり表れています。機能ごとのルートも、後の章で見るように、ただの`Routes`配列として定義し、`loadChildren`で読み込むだけです。モジュールという中間層が消えたぶん、ルート定義とその読み込みの関係が、まっすぐ見通せるようになりました。

:::message
既存プロジェクトの`RouterModule`ベースの設定も、そのまま動作します。Standaloneへの移行スキマティクス（第7章）を使うと、`provideRouter()`ベースへの変換も進められます。
:::

## よくあるつまずき

- **`RouterOutlet`の置き忘れ**: Routesを定義してもComponentが表示されないときは、テンプレートに`<router-outlet />`があるかを確認します。表示する場所がなければ、何も現れません。
- **`imports`への宣言忘れ**: `RouterOutlet`・`routerLink`をテンプレートで使うには、Componentの`imports`への宣言が必要です。
- **ワイルドカードの位置**: `path: '**'`は、必ず配列の最後に置きます。前方一致で先に評価されるため、途中に置くと、以降のルートが無視されます。
- **ルートの順序**: Routesは上から順に照合され、最初に一致したものが使われます。より具体的なパスを先に、汎用的なパスを後に並べるのが原則です。順序を誤ると、意図しないルートに一致してしまいます。
- **`routerLink`と`href`の混同**: 内部の遷移に`href`を使うと、ページ全体が再読み込みされ、SPAの利点が失われます。アプリ内の遷移は必ず`routerLink`を使います。

## まとめ

- Routesは、`path`と`component`の対応でルートを定義する配列です
- `provideRouter(routes)`を`app.config.ts`に加えて、ルーティングを登録します
- `with〜`関数で、入力への結びつけや先読みなどの機能を追加できます
- 表示するComponentは、`<router-outlet />`の位置に差し込まれます
- **新規開発では`provideRouter()`を使うのが現在の標準です。`RouterModule`は既存コードの理解のために押さえます**

次章では、実際にページを遷移する方法と、URLに含まれるパラメーターの受け取り方を学びます。
