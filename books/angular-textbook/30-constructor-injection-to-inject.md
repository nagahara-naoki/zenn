---
title: "Constructor Injectionからinject()へ"
---

前章で、依存を注入してもらうというDIの考え方を学びました。この章では、その注入を実際に受け取る書き方に踏み込みます。Angularには、依存を受け取る方法が2つあります。ひとつは、コンストラクターの引数で受け取るConstructor Injection。もうひとつが、現在の標準である`inject()`関数です。

`inject()`は、Angular 14（2022年）で導入されました。それ以前はConstructor Injectionが唯一の方法で、既存コードの多くはこの形で書かれています。両者は同じDIの仕組みを使っており、結果は変わりません。しかし、書き味と柔軟性に違いがあります。この章では、両方を比較しながら、なぜ現在は`inject()`が推奨されるのかを理解します。

:::message
**この章で学ぶこと**

- Constructor Injectionによる依存の受け取り
- `inject()`関数による依存の受け取り
- 両者の違いと、`inject()`の利点
- 注入の細かな制御（`optional`など）
:::

## Constructor Injectionという書き方

まず、旧来のConstructor Injectionを見ます。依存を、コンストラクターの引数として宣言する方法です。

```ts:旧来の書き方（Constructor Injection）
import { Component } from '@angular/core';
import { ProductService } from './product';

@Component({ selector: 'app-product-list-page', template: `...` })
export class ProductListPage {
  constructor(private readonly service: ProductService) {}

  load(): void {
    this.service.getProducts();
  }
}
```

コンストラクターの引数に`private readonly service: ProductService`と書くと、Angularが`ProductService`を注入し、`this.service`として使えるようにします。引数に`private`などの修飾子を付けることで、そのままクラスのプロパティになる、というTypeScriptの機能を利用しています。

この書き方は長く標準でしたが、いくつかの不便がありました。依存が増えるとコンストラクターの引数が長くなること、クラスを継承したときに親のコンストラクター引数をすべて引き継いで書き直す必要があること、そして関数の中では使えないことです。

## inject()という書き方

同じ依存の受け取りを、`inject()`で書くと、次のようになります。

```ts:src/app/product-list-page.ts
import { Component, inject } from '@angular/core';
import { ProductService } from './product';

@Component({ selector: 'app-product-list-page', template: `...` })
export class ProductListPage {
  private readonly service = inject(ProductService);

  load(): void {
    this.service.getProducts();
  }
}
```

`inject(ProductService)`を、フィールドの初期化子として書きます。コンストラクターは不要です。受け取った依存は、ふつうのフィールドとして`this.service`で使えます。前章で触れたように、`inject()`は注入コンテキスト、すなわちクラスの初期化のタイミングで呼び出します。

見た目の違いはわずかですが、この違いが、いくつもの利点につながります。

## inject()の利点

`inject()`が現在推奨される理由は、Constructor Injectionの不便を解消するからです。

**コンストラクターが不要になる**。依存を受け取るためだけにコンストラクターを書く必要がなくなり、フィールドの宣言として自然に並べられます。依存が増えても、引数リストが長大になることはありません。

**継承が楽になる**。クラスを継承するとき、Constructor Injectionでは、親が受け取っている依存を子のコンストラクターでも書き、`super()`へ渡す必要がありました。`inject()`なら、親クラスの中で`inject()`しているため、子はコンストラクターに手を加えなくて済みます。

**関数の中でも使える**。これが特に大きな違いです。`inject()`は、クラスだけでなく、Angularが提供する関数の中でも呼べます。第7部で学ぶ関数型のRoute Guardや、第9部で学ぶ関数型のInterceptorは、この性質の上に成り立っています。たとえば、Guardを次のように関数として書き、その中で依存を注入できます。

```ts:src/app/auth-guard.ts
import { inject } from '@angular/core';
import { CanActivateFn } from '@angular/router';

export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService); // 関数の中で注入できる
  return auth.isLoggedIn();
};
```

Constructor Injectionは、コンストラクターを持つクラスでしか使えないため、こうした関数の中では依存を受け取れませんでした。`inject()`は、DIをクラスの外へ解き放ち、より柔軟な書き方を可能にしたのです。

**再利用しやすい**。複数のComponentで共通する注入と初期化の処理を、ひとつの関数にまとめて切り出せます。たとえば、`inject()`を内部で使うヘルパー関数を作り、各Componentのフィールド初期化子から呼ぶ、といった書き方ができます。これはConstructor Injectionでは難しいことでした。

## 新旧を並べて比べる

同じ依存を受け取るコードを、両者で並べます。

```ts:旧来の書き方（Constructor Injection）
export class UserPage {
  constructor(
    private readonly userService: UserService,
    private readonly logger: Logger,
  ) {}
}
```

```ts:src/app/user-page.ts（現在の書き方）
export class UserPage {
  private readonly userService = inject(UserService);
  private readonly logger = inject(Logger);
}
```

依存が増えたときの見やすさの差が分かります。`inject()`版は、各依存が独立した一行として並び、追加や削除も一行の操作で済みます。コンストラクターの引数リストを編集する必要がありません。

型の面でも`inject()`は素直です。Constructor Injectionは、引数の型注釈をAngularが読み取ってトークンを決める仕組みで、この解決はTypeScriptのコンパイル時の情報に依存していました。`inject(ProductService)`は、トークンを値として直接渡すため、何を注入しようとしているのかがコードから明快に読み取れます。戻り値の型も、渡したトークンから自動で決まるため、`this.service`の型は`ProductService`だと、型注釈を書かずとも正しく推論されます。宣言の簡潔さと型の正確さが、両立しているのです。

## 注入を細かく制御する

`inject()`には、注入のふるまいを調整するオプションを渡せます。よく使うのが、依存が見つからないときにエラーにせず`null`を許す`optional`です。

```ts
// 見つからなければ null（エラーにしない）
private readonly config = inject(AppConfig, { optional: true });
```

ほかにも、探索する範囲を制御するオプションがあります。

- **`optional`**: 見つからなければ`null`を返す（エラーにしない）
- **`self`**: 自身のInjectorだけを探す
- **`skipSelf`**: 自身を飛ばして、親のInjectorから探す
- **`host`**: 探索をホストの境界で止める

これらは、次章で学ぶInjectorの階層と関わります。ふだんは`inject(ProductService)`と書くだけで十分ですが、こうした細かな制御ができることも知っておくと、複雑な場面で役立ちます。旧来は、`@Optional()`・`@Self()`・`@SkipSelf()`といったデコレーターをコンストラクター引数に添えて、同じことを実現していました。`inject()`では、これらをオプションのオブジェクトとして渡せます。

## よくあるつまずき

- **注入コンテキストの外で`inject()`を呼ぶ**: `inject()`は、クラスの初期化時（フィールド初期化子やコンストラクター）に呼びます。ボタンクリックのハンドラーなど、初期化が終わった後の処理の中で呼ぶと、エラーになります。依存は初期化時に受け取り、フィールドに保持しておきます。
- **Constructor Injectionと混在させて混乱する**: 1つのクラスで両方を使うこともできますが、統一したほうが読みやすくなります。新規のコードは`inject()`に揃えるのがよいでしょう。
- **移行を急ぎすぎる**: 既存のConstructor Injectionは、そのままでも問題なく動きます。公式の移行スキマティクスもあるため、慌てて手作業で書き換える必要はありません。

:::message
既存プロジェクトのConstructor Injectionを`inject()`へ書き換える、公式の移行スキマティクスが用意されています。次のコマンドで実行でき、コンストラクター引数を`inject()`のフィールド宣言へ自動で変換します。

```bash
ng generate @angular/core:inject
```

差分を確認しながら段階的に進められるため、大きなプロジェクトでも安全に移行できます。
:::

## まとめ

- 依存を受け取る方法には、Constructor Injectionと`inject()`があります
- `inject()`はAngular 14で導入され、フィールド初期化子として依存を受け取ります
- `inject()`は、コンストラクター不要・継承が楽・関数内でも使える、という利点があります
- 関数型のGuardやInterceptorは、`inject()`があって初めて成り立ちます
- **新規開発では`inject()`を使うのが現在の標準です。Constructor Injectionは既存コードの理解のために押さえます**

次章では、注入されるServiceがどこで作られ、どの範囲で共有されるのかという、ProviderとInjectorの階層を学びます。
