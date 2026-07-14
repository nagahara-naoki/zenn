---
title: "Route GuardとResolverの新旧比較"
---

第7部の締めくくりとして、ルーティングに関門を設ける2つの仕組み、GuardとResolverを学びます。Guardは「このページに入ってよいか」を判断する門番、Resolverは「このページを表示する前にデータをそろえる」下ごしらえ役です。どちらも、遷移のタイミングに割り込んで処理を行います。

これらの書き方も、モダンAngularで大きく変わりました。かつてはクラスとインターフェースで実装していましたが、現在は関数で書きます。この関数型のGuard・Resolverは、第24章で学んだ`inject()`があってこそ成り立つものです。この章では、関数型を主に、旧来のクラス型と比較しながら、アクセス制御と事前データ取得を学びます。

:::message
**この章で学ぶこと**

- Route Guardによるアクセス制御
- 関数型Guardの書き方と種類
- Resolverによる事前データ取得
- 旧来のクラス型との違い
:::

## Route Guardとは

Route Guardは、ルートへの遷移を許可するか、拒否するかを判断する仕組みです。もっとも多い用途は、認証によるアクセス制御です。「ログインしていない利用者は、マイページに入れない」といった制御を、Guardで実現します。

Guardは、遷移が起きようとするタイミングで呼ばれます。Guardが`true`を返せば遷移が進み、`false`を返せば遷移が止まります。あるいは、`UrlTree`（別のURLへの指示）を返して、ログインページへ振り向けることもできます。門番が通行の可否を判断し、必要なら別の場所へ案内する、というイメージです。

## 関数型Guardを書く

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

## Guardの種類

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

## Resolverで事前にデータを用意する

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

## 旧来のクラス型との比較

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

## よくあるつまずき

- **Guardの中で`inject()`を関数の外に置く**: 関数型Guardの`inject()`は、関数本体の中で呼びます。Guardが呼ばれる文脈は注入コンテキストとして扱われるため、その中で依存を受け取れます。
- **`false`とリダイレクトを混同する**: 単に`false`を返すと、遷移が止まるだけで、その場にとどまります。ログインページなどへ誘導したいなら、`UrlTree`を返します。「拒否」と「別の場所へ案内」は別物です。
- **Resolverで重いデータを待たせる**: 取得の遅いデータをResolverに入れると、遷移全体が待たされ、操作が固まったように見えます。時間のかかるものは、ページ表示後に読み込むほうが体感がよい場合があります。
- **Guardの副作用に頼る**: Guardは可否の判断に徹します。Guardの中でデータを書き換えるなどの副作用を持たせると、遷移が中断されたときに整合性が崩れます。

## まとめ

- Guardは、ルートへの遷移の可否を判断する門番の仕組みです
- 関数型Guardは`inject()`で依存を受け取り、`canActivate`などに指定します
- Guardには`CanActivateFn`・`CanDeactivateFn`・`CanMatchFn`などの種類があります
- Resolverは、遷移の前にデータを用意する仕組みで、`ResolveFn`で書きます
- **新規開発では関数型のGuard・Resolverを使うのが現在の標準です。クラス型は既存コードの理解のために押さえます**

以上で第7部は終わりです。次の第8部では、非同期処理を扱うための強力な仕組みであるRxJSを学びます。
