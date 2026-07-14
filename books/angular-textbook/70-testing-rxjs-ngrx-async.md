---
title: "第58章 RxJS・NgRx・非同期処理のテスト戦略"
---

前章で、テストの基礎を学びました。しかし、実際のアプリケーションには、テストが難しい部分があります。時間をかけて値が流れるRxJS、通信を伴うHTTP、そしてAction・Reducer・EffectsからなるNgRxです。これらは非同期であったり、複数の要素が絡んだりするため、単純な「入力と出力」の確認だけでは扱えません。

この章では、こうした難しい対象を、どうテストするかの戦略を学びます。非同期処理の完了を待つ方法、通信をテスト用に差し替える方法、そしてNgRxの各要素をテストする勘所を扱います。ここでも、これまで学んだ「純粋な関数はテストしやすい」「依存は差し替えられる」という原則が、指針になります。

:::message
**この章で学ぶこと**

- Observableのテスト
- 非同期処理を待つ方法
- HTTP通信のテスト
- NgRxの各要素のテスト
:::

## Observableをテストする

Observableのテストの基本は、購読して、流れてきた値を確かめることです。同期的に値を流すObservableなら、購読の中で`expect`を書けます。

```ts:src/app/example.spec.ts
import { of } from 'rxjs';
import { map } from 'rxjs';

it('値を2倍にする', () => {
  const result: number[] = [];
  of(1, 2, 3).pipe(map((n) => n * 2)).subscribe((v) => result.push(v));
  expect(result).toEqual([2, 4, 6]);
});
```

`of`は同期的に値を流すため、`subscribe`の直後に結果を確かめられます。流れてきた値を配列に集め、期待と比較します。より複雑な、時間の絡むObservableのテストには、marble testing（マーブルテスト）という、時間の流れを図のように記述する高度な手法もあります。ただし、多くの場面では、この「購読して集めて確かめる」基本形で十分です。

## 非同期処理を待つ

`setTimeout`やPromiseのように、時間を要する非同期処理は、テストの中でその完了を待つ必要があります。Angularには、`fakeAsync`と`tick`という仕組みがあります。

```ts:src/app/timer.spec.ts
import { fakeAsync, tick } from '@angular/core/testing';

it('1秒後に値が更新される', fakeAsync(() => {
  const service = new TimerService();
  service.start();

  tick(1000); // 仮想的に1秒進める

  expect(service.value()).toBe(1);
}));
```

`fakeAsync`で囲んだテストの中では、時間を仮想的に操作できます。`tick(1000)`で「1秒経過した」ことにすると、その間に予約された非同期処理が実行されます。実際に1秒待つ必要がないため、テストは一瞬で終わります。時間に依存する処理を、確実に、すばやくテストできるのが利点です。Signalの非同期更新や、`debounceTime`を含む処理のテストに役立ちます。

## HTTP通信をテストする

通信を含むServiceのテストで、実際にサーバーへ通信してしまうと、テストが不安定になります。そこで、通信をテスト用に差し替えます。Angularは、`provideHttpClientTesting`と`HttpTestingController`を提供します。

```ts:src/app/product.spec.ts
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';

it('商品一覧を取得する', () => {
  TestBed.configureTestingModule({
    providers: [provideHttpClient(), provideHttpClientTesting()],
  });
  const service = TestBed.inject(ProductService);
  const httpMock = TestBed.inject(HttpTestingController);

  let result: Product[] = [];
  service.getProducts().subscribe((products) => (result = products));

  // 通信を横取りし、テスト用の応答を返す
  const req = httpMock.expectOne('/api/products');
  req.flush([{ id: '1', name: '本' }]);

  expect(result.length).toBe(1);
});
```

`provideHttpClientTesting`により、実際の通信は行われず、`HttpTestingController`が横取りします。`expectOne`で「この通信が起きたはずだ」と確認し、`flush`でテスト用の応答を返します。これにより、サーバーなしで、通信の成功・失敗を含めた挙動を確かめられます。第23章のDIによる差し替えの、通信版だと考えるとよいでしょう。

## NgRxをテストする

NgRxは、要素ごとに責務が分かれているため、実はテストしやすい設計です。要素ごとに、テストの勘所を見ていきます。

**Reducer**は、純粋な関数です。第52章で学んだとおり、同じ状態とActionからは、常に同じ状態を返します。そのため、テストはもっとも単純です。状態とActionを与え、返る状態を確かめるだけです。

```ts:src/app/counter.reducer.spec.ts
it('incrementで count が1増える', () => {
  const state = { count: 0 };
  const next = counterReducer(state, increment());
  expect(next.count).toBe(1);
});
```

**Selector**も、純粋な関数としてテストできます。状態を渡し、取り出される値を確かめます。`createSelector`が作るSelectorは、状態を引数に取る関数として、直接呼び出せます。

**Effects**は、非同期を含むため、少し複雑です。Actionの流れを与え、期待するActionが発行されるかを確かめます。通信を伴う場合は、前述のHTTPテストの手法と組み合わせます。Effectsのテストは、「あるActionを流したら、通信の結果に応じて、成功Actionまたは失敗Actionが発行される」ことを確認するのが基本です。

このように、Reduxパターンの「純粋な関数への分離」は、テスト容易性という形でも報われます。状態変更ロジックがReducerに、副作用がEffectsに分かれているため、それぞれを独立してテストできるのです。

## テストしやすさは設計のよさ

この章と前章を通して見えてくるのは、「テストしやすいコードは、設計のよいコードである」ということです。純粋な関数、依存の注入、責務の分離。これらは、本書が繰り返し説いてきた設計原則であり、同時に、テストを容易にする条件でもあります。

もしテストが書きにくいと感じたら、それは設計を見直す合図かもしれません。Componentにロジックを詰め込みすぎていないか、依存を`new`で直接作っていないか、副作用と純粋な処理が混ざっていないか。テストの書きにくさは、設計の歪みを映す鏡です。テストを意識することは、よい設計へと導く力にもなります。テストは、品質を守るだけでなく、設計を磨く道具でもあるのです。

## SignalStoreのテスト

第55章で学んだNgRx SignalStoreも、Signalベースであるため、テストしやすい部類です。Storeを注入（またはテスト用に生成）し、メソッドを呼んで、状態のSignalを確認します。

```ts:src/app/counter.store.spec.ts
it('incrementで count が増える', () => {
  TestBed.configureTestingModule({});
  const store = TestBed.inject(CounterStore);

  store.increment();

  expect(store.count()).toBe(1);
});
```

`store.increment()`を呼び、`store.count()`で結果を確かめるだけです。ActionをdispatchしてSelectorで読む従来のNgRx Storeより、テストの記述は素直になります。状態をSignalで扱うことの利点が、テストの場面でも一貫して表れます。非同期を含むメソッド（`rxMethod`）のテストは、本章で学んだHTTPの差し替えや`fakeAsync`と組み合わせます。

## テストの粒度を考える

テストには粒度があります。ひとつの関数を対象とする細かいテストから、複数の部品の連携を確かめるテスト、アプリ全体を通すE2Eテストまで、対象の広さが異なります。すべてを同じ密度で書くのは非効率です。

一般には、細かい単体テストを土台に多く書き、結合テストをその上に、E2Eテストは要所に絞る、という配分が推奨されます。単体テストは速く、安定し、原因の特定も容易です。E2Eテストは、利用者の視点で全体を確かめられますが、遅く、壊れやすく、原因の特定も難しくなります。この配分の考え方は「テストピラミッド」として知られます。本書では詳しく立ち入りませんが、「速く安定したテストを多く、遅く壊れやすいテストは少なく」という原則を、頭に置いておくとよいでしょう。

## よくあるつまずき

- **`fakeAsync`の外で非同期を待つ**: 時間の絡む処理は、`fakeAsync`と`tick`で仮想的に進めます。実時間を待つと、テストが遅く不安定になります。
- **`flush`や`expectOne`を忘れる**: HTTPテストで、`expectOne`と`flush`を呼ばないと、通信が完了せず、期待した結果になりません。通信の横取りと応答を、忘れずに書きます。
- **Effectsのエラー処理をテストしない**: 通信の成功だけでなく、失敗時に失敗Actionが発行されるかも確認します。エラー経路こそ、テストの価値が高い部分です。
- **実装の詳細をテストする**: 内部の細かい実装をテストすると、リファクタリングのたびに壊れます。外から見た振る舞い（入力と出力）を対象にします。

## まとめ

- Observableは、購読して流れてきた値を集め、期待と比較してテストします
- `fakeAsync`と`tick`で、時間に依存する非同期処理を、待たずにテストできます
- HTTP通信は、`provideHttpClientTesting`で差し替え、サーバーなしでテストします
- NgRxは、純粋なReducer・Selectorが特にテストしやすく、Effectsは非同期として扱います
- テストしやすさは設計のよさの表れで、テストは設計を磨く道具にもなります

次章では、アプリケーションを守るセキュリティ、とくにXSS対策とサニタイゼーションを学びます。
