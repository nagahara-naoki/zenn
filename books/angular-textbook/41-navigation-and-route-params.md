---
title: "ページ遷移とルートパラメーター"
---

前章で、ルーティングの基本設定を整えました。この章では、実際にページを遷移する方法と、URLに埋め込まれた値、すなわちルートパラメーターの受け取り方を学びます。

「商品一覧から、特定の商品の詳細ページへ移る」という動きを考えてみましょう。ここには2つの要素があります。ひとつは、リンクやボタンで詳細ページへ移ること。もうひとつは、「どの商品か」という情報をURLで伝え、詳細ページ側でそれを受け取ることです。前者が画面遷移、後者がルートパラメーターです。この章では、両方を扱い、モダンAngularらしい受け取り方まで踏み込みます。

:::message
**この章で学ぶこと**

- `routerLink`によるリンク遷移
- `routerLinkActive`による現在地の強調
- プログラムからの遷移（`Router`の利用）
- ルートパラメーターの定義と受け取り
:::

## routerLinkでリンク遷移する

もっとも基本的な遷移は、`routerLink`によるリンクです。前章でも触れたとおり、`<a>`タグに`routerLink`を付けます。

```html
<a routerLink="/about">概要ページへ</a>
```

`href`ではなく`routerLink`を使う点が肝心です。`href`で書くと、ブラウザはサーバーへ新しいページを要求し、SPAの利点が失われます。`routerLink`なら、Routerが遷移を引き受け、再読み込みなしで画面を切り替えます。

遷移先が固定ではなく、値を組み込む場合は、配列の形で書きます。

```html
<a [routerLink]="['/products', product.id]">詳細を見る</a>
```

この書き方は、`product.id`が`42`なら`/products/42`というURLを組み立てます。配列の各要素が、URLのセグメント（区切り）に対応します。値を含むリンクは、この配列形式で書くのが基本です。角括弧`[routerLink]`とすることで、中身が式として評価される点にも注意してください。

## routerLinkActiveで現在地を示す

ナビゲーションでは、「いまどのページにいるか」を見た目で示したいことがよくあります。そのための機能が`routerLinkActive`です。現在のURLがそのリンク先と一致するとき、指定したCSSクラスを付与します。

```html
<a routerLink="/about" routerLinkActive="active">概要</a>
```

`/about`を表示しているあいだ、このリンクには`active`クラスが付きます。あとはCSSで`.active`のスタイルを定義すれば、現在地のリンクだけを強調できます。ナビゲーションメニューの「選択中」の表現に、そのまま使えます。

## プログラムから遷移する

リンクだけでなく、処理の中からページを遷移させたいこともあります。「保存ボタンを押したら、保存してから一覧へ戻る」といった場面です。このときは、`Router`を注入し、その`navigate`メソッドを使います。

```ts:src/app/product-edit.ts
import { Component, inject } from '@angular/core';
import { Router } from '@angular/router';

@Component({ selector: 'app-product-edit', template: `...` })
export class ProductEdit {
  private readonly router = inject(Router);

  async save(): Promise<void> {
    // 保存処理（省略）
    await this.router.navigate(['/products']);
  }
}
```

`this.router.navigate(['/products'])`で、一覧ページへ遷移します。引数は`routerLink`と同じ配列形式です。値を含めるなら`['/products', id]`のように書きます。URL文字列をそのまま渡したい場合は、`navigateByUrl('/products')`も使えます。処理の流れの中で遷移を制御したいときは、この`Router`による方法を用います。

## ルートパラメーターを定義する

次に、URLに値を埋め込む「ルートパラメーター」を扱います。商品詳細ページのように、「同じ形のページで、対象だけが違う」場面で使います。まず、ルート定義でパラメーターの位置を`:名前`で示します。

```ts:src/app/app.routes.ts
export const routes: Routes = [
  { path: 'products', component: ProductList },
  { path: 'products/:id', component: ProductDetail },
];
```

`products/:id`の`:id`が、パラメーターです。`/products/42`なら`id`が`42`に、`/products/99`なら`id`が`99`になります。ひとつのルート定義で、あらゆる商品の詳細ページを表せるわけです。あとは、詳細ページ側でこの`id`を受け取れば、対象の商品を特定できます。

## パラメーターをinputで受け取る

受け取り方には、新旧2つの方法があります。まず、モダンAngularらしい方法から見ます。前章で`provideRouter(routes, withComponentInputBinding())`を設定したことを思い出してください。この`withComponentInputBinding()`があると、ルートパラメーターが、Componentの`input()`へ自動で結びつきます。

```ts:src/app/product-detail.ts
import { Component, computed, inject, input } from '@angular/core';
import { ProductService } from './product';

@Component({
  selector: 'app-product-detail',
  template: `<h1>{{ product()?.name }}</h1>`,
})
export class ProductDetail {
  private readonly service = inject(ProductService);

  // パラメーター :id が、この input に結びつく
  readonly id = input.required<string>();
  protected readonly product = computed(() => this.service.findById(this.id()));
}
```

`id = input.required<string>()`と宣言するだけで、URLの`:id`の値が`id`に入ります。パラメーター名（`id`）と入力名（`id`）を合わせるのがポイントです。しかも入力はSignalなので、`computed()`と組み合わせれば、URLが変わるたびに商品が自動で切り替わります。第18章で学んだ入力の仕組みが、ルートパラメーターにもそのまま活きるのです。購読のコードは一切要りません。これが、現在もっとも簡潔な受け取り方です。

## パラメーターをActivatedRouteで受け取る

もうひとつの方法が、`ActivatedRoute`を使うものです。`withComponentInputBinding()`が登場する前から使われてきた、従来からの方法です。

```ts:src/app/product-detail.ts（ActivatedRouteを使う方法）
import { Component, inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

@Component({ selector: 'app-product-detail', template: `...` })
export class ProductDetail {
  private readonly route = inject(ActivatedRoute);

  constructor() {
    this.route.paramMap.subscribe((params) => {
      const id = params.get('id');
      // idを使った処理
    });
  }
}
```

`ActivatedRoute`は、現在のルートに関する情報を持つオブジェクトです。その`paramMap`を購読すると、パラメーターの変化を受け取れます。`paramMap`がObservableである理由は、同じComponentのまま`id`だけが変わる遷移（`/products/42`から`/products/99`へ）に対応するためです。この場合Componentは再生成されず、`paramMap`が新しい値を流します。

`withComponentInputBinding()`による`input()`方式は、この`ActivatedRoute`の購読を、内部で肩代わりしてくれるものだと考えるとよいでしょう。新規に書くなら`input()`方式が簡潔ですが、既存コードには`ActivatedRoute`が多く登場するため、両方を知っておくことが大切です。

## クエリパラメーターとその他の情報

URLには、`:id`のようなパスパラメーターのほかに、`?keyword=angular&page=2`のようなクエリパラメーターもあります。検索条件やページ番号など、任意の付加情報を表すのに使います。これらも、`ActivatedRoute`の`queryParamMap`や、`withComponentInputBinding()`による入力で受け取れます。

```html
<!-- クエリパラメーター付きのリンク -->
<a [routerLink]="['/search']" [queryParams]="{ keyword: 'angular' }">検索</a>
```

パスパラメーターが「どのリソースか」を表すのに対し、クエリパラメーターは「どう絞り込むか・並べるか」といった、表示の調整に向きます。用途に応じて使い分けます。たとえば、商品`42`の詳細はパスパラメーター（`/products/42`）で、その一覧の検索条件やページ番号はクエリパラメーター（`/products?keyword=本&page=2`）で表す、という具合です。パスパラメーターは「そのページが何を指すか」の一部であり、クエリパラメーターは「同じページの見せ方の違い」だと捉えると、迷いにくくなります。

## まとめ

- ページ遷移は、テンプレートでは`routerLink`、処理内では`Router`の`navigate`で行います
- 値を含むリンクは`[routerLink]="['/products', id]"`のように配列で書きます
- `routerLinkActive`で、現在地のリンクを強調できます
- ルートパラメーターは`:id`で定義し、`withComponentInputBinding()`で`input()`に結びつきます
- 従来は`ActivatedRoute`の`paramMap`を購読して受け取っており、既存コードで多く見られます

次章では、ページの中にページを持つネストしたRouteと、共通レイアウトの設計を学びます。
