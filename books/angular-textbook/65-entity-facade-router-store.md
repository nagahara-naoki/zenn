---
title: "Entity・Facade・Router Storeによる実務設計"
---

前章までで、NgRxの基本要素（Action・Reducer・Selector・Effects）を学びました。この章では、それらを実務で使うときに役立つ、いくつかの設計パターンを扱います。NgRxを大規模に使うと、定型的なコードが増えたり、Componentからの利用が煩雑になったりします。これらを整理するのが、Entity・Facade・Router Storeといった仕組みです。

これらは、NgRxを「使える」段階から、「うまく設計できる」段階へ進むための道具立てです。すべてを常に使う必要はありませんが、それぞれが何を解決するのかを知っておくと、大規模なアプリの状態管理を、より見通しよく組み立てられます。この章では、それぞれの役割と、いつ使うべきかを解説します。

:::message
**この章で学ぶこと**

- `@ngrx/entity`によるコレクション管理
- Facadeパターンによる利用の簡素化
- `@ngrx/router-store`によるルーティング状態の管理
- これらのパターンの使いどころ
:::

## Entityによるコレクション管理

アプリでは、「商品の一覧」「ユーザーの一覧」のように、同じ種類のデータの集まり（コレクション）を扱うことがよくあります。こうしたコレクションを、配列で持つと、「特定のIDのものを更新する」「1件を削除する」といった操作を、そのつど自分で書くことになります。これは定型的で、間違いも起きやすい作業です。

`@ngrx/entity`は、このコレクション管理を助ける仕組みです。`createEntityAdapter`が、追加・更新・削除といった定型操作を提供します。

```ts:src/app/product.reducer.ts
import { createEntityAdapter, EntityState } from '@ngrx/entity';

export interface ProductState extends EntityState<Product> {
  loading: boolean;
}

const adapter = createEntityAdapter<Product>();
const initialState: ProductState = adapter.getInitialState({ loading: false });

export const productReducer = createReducer(
  initialState,
  on(loadProductsSuccess, (state, { products }) =>
    adapter.setAll(products, { ...state, loading: false }),
  ),
  on(updateProduct, (state, { product }) =>
    adapter.updateOne({ id: product.id, changes: product }, state),
  ),
);
```

`adapter.setAll`や`adapter.updateOne`が、コレクションの操作を引き受けます。データは、内部的にはIDをキーにした効率的な形で保持されます。自分で配列を操作するより、簡潔で、間違いが減ります。コレクションを扱うNgRxの状態には、Entityを使うのが定石です。あわせて、一覧や1件を取り出すSelectorも、Adapterが提供してくれます。

## Facadeパターンによる簡素化

NgRxをそのまま使うと、Componentは`Store`を注入し、Selectorで読み取り、Actionをdispatchする、という一連を自分で書きます。これは、NgRxの詳細（どんなActionやSelectorがあるか）を、Componentが知っていることを意味します。Componentが多いと、この知識があちこちに散らばります。

Facade（ファサード）パターンは、この複雑さを、ひとつのServiceの裏に隠します。Facadeは、NgRxのStoreをラップし、Componentにわかりやすいメソッドとプロパティを提供します。

```ts:src/app/product.facade.ts
@Injectable({ providedIn: 'root' })
export class ProductFacade {
  private readonly store = inject(Store);

  // 状態は、わかりやすい名前のSignalで公開する
  readonly products = this.store.selectSignal(selectAllProducts);
  readonly loading = this.store.selectSignal(selectLoading);

  // 操作は、わかりやすいメソッドで公開する
  load(): void {
    this.store.dispatch(loadProducts());
  }
}
```

Componentは、`ProductFacade`を注入し、`facade.products()`で状態を読み、`facade.load()`で操作します。NgRxのActionやSelectorを、直接は知りません。これにより、Componentが単純になり、NgRxの実装を変えても、Facadeの中だけを直せば済みます。ただし、Facadeは層を1つ増やすため、小規模なうちは過剰になることもあります。規模とチームの状況を見て採り入れます。

## Router Storeによるルーティング状態

第7部で、現在のページや検索条件はURLで表す、と学びました。`@ngrx/router-store`は、そのルーティングの状態を、NgRxのStoreと結びつける仕組みです。これを使うと、現在のURLやルートパラメーターを、Selectorで読み取れるようになります。

```ts:src/app/app.config.ts
import { provideRouterStore } from '@ngrx/router-store';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore({ /* ... */ }),
    provideRouterStore(), // ルーティング状態をStoreに統合
  ],
};
```

これにより、「現在のルートパラメーターに応じたデータを、Selectorで組み立てる」といった設計ができます。ルーティングの状態と、アプリの状態を、ひとつのStoreの上で一貫して扱えるのが利点です。ただし、多くの場合は第7部で学んだRouterの標準機能で十分であり、Router Storeが必要になるのは、ルーティング状態を他の状態と密に組み合わせたい、限られた場面です。

## パターンの使いどころ

これらのパターンは、いずれも「必要になったら使う」ものです。最初からすべてを導入する必要はありません。判断の目安を示します。

- **Entity**: 同じ種類のデータのコレクションを扱うなら、ほぼ常に有用です。早い段階から使う価値があります。
- **Facade**: Componentが多く、NgRxの詳細を隠したいときに有用です。小規模では過剰になりえます。
- **Router Store**: ルーティング状態を他の状態と密に組み合わせる、特定の要件があるときに使います。

第48章の「小さく始め、必要に応じて育てる」という原則は、NgRxの内部でも当てはまります。基本のAction・Reducer・Selectorから始め、コレクションが増えたらEntityを、利用が煩雑になったらFacadeを、と段階的に採り入れるのが、健全な進め方です。パターンは、複雑さに対処するための道具であって、複雑さを増やすために使うものではありません。

## ファイル構成の指針

NgRxを使うと、ひとつの機能に対して、Action・Reducer・Selector・Effectsと、複数のファイルができます。これらをどう配置するかも、実務では大切です。一般には、機能ごとのフォルダにまとめるのがよいでしょう。

```text
src/app/product/
├── product.actions.ts
├── product.reducer.ts
├── product.selectors.ts
├── product.effects.ts
└── product.facade.ts
```

このように、`product`という機能に関するNgRxのファイルを、ひとつのフォルダに集めます。第35章で学んだFeature単位の分割と、同じ発想です。機能ごとにまとまっていれば、その機能の状態管理の全体を、一か所で把握できます。遅延読み込みする機能なら、その機能の状態を`provideState`で、機能のルート定義とともに登録します。状態管理のコードも、アプリケーションのFeature構造に沿って整理するのが、大規模開発では重要です。

## よくあるつまずき

- **コレクションを素の配列で持つ**: 同種のデータの集まりを配列で持って手作業で更新すると、間違いが増えます。`@ngrx/entity`を使い、定型操作を任せます。
- **最初からFacadeを作る**: 小規模なうちからFacadeを挟むと、層が増えるだけで利点が薄いことがあります。Componentが増え、NgRxの詳細を隠したくなってから導入します。
- **何でもRouter Storeに載せる**: ルーティング状態は、多くの場合Routerの標準機能で足ります。他の状態と密に組み合わせる明確な理由があるときだけ、Router Storeを使います。
- **ファイルを役割別に分けすぎる**: すべてのActionを1ファイル、すべてのReducerを別ファイル、と役割で束ねると、機能をまたいで探し回ることになります。機能ごとにまとめます。

## SignalStoreとの関係

ここで紹介したEntityやFacadeは、主に従来のNgRx Store（Reduxパターン）を前提としたものです。次章で扱うNgRx SignalStoreでは、状況が少し変わります。SignalStoreは、状態とメソッドをひとつのStoreにまとめる作りのため、Facadeのような「Storeを隠す層」を、多くの場合そもそも必要としません。Store自体が、Componentにとって使いやすいインターフェースになっているからです。

Entityに相当するコレクション管理も、SignalStore向けの部品として提供されています。つまり、ここで学んだパターンの「考え方」は、SignalStoreでも通じますが、その「実現手段」は、選んだStoreの種類によって変わります。パターンの本質（コレクションを効率的に扱う、詳細を隠して使いやすくする）を理解しておけば、どちらのStoreでも応用できます。次章で、その2つのStoreの選択を、あらためて整理します。

## まとめ

- `@ngrx/entity`は、コレクションの追加・更新・削除を`createEntityAdapter`で簡潔に扱います
- Facadeパターンは、NgRxの詳細をServiceの裏に隠し、Componentを単純にします
- `@ngrx/router-store`は、ルーティング状態をStoreに統合します
- これらは必要になったときに採り入れる道具で、最初からすべては要りません
- 基本から始め、複雑さに応じてパターンを段階的に足すのが健全な進め方です

次章では、この部の締めくくりとして、NgRx SignalStoreと従来のNgRx Storeの使い分けを学びます。
