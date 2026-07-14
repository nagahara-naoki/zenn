---
title: "第57章 テストの基礎とComponent・Serviceのテスト — TestBed・Harness・Vitest"
---

アプリケーションを長く安全に育てるには、テストが欠かせません。テストは、コードが期待どおり動くことを自動で確認する仕組みです。手作業での確認と違い、何度でも、すばやく、もれなく実行できます。変更のたびにテストを走らせれば、それまで動いていた機能を壊していないかを、すぐに確かめられます。

この章では、Angularにおけるテストの基礎を学びます。テストの土台となるTestBed、Componentのテスト、Serviceのテスト、そしてComponent Harnessという道具を扱います。本書が基準とするv22では、テストの実行環境がVitestになりました。この新しい既定にも触れながら、テストの考え方と書き方を身につけます。

:::message
**この章で学ぶこと**

- Angularのテスト環境（Vitest）
- Serviceのテスト
- TestBedによるComponentのテスト
- Component Harnessによる安定したテスト
:::

## テストの種類と実行環境

テストには、いくつかの種類があります。ひとつの関数やクラスを対象とする単体テスト（ユニットテスト）、複数の部品の連携を確かめる結合テスト、アプリ全体を通しで動かすE2E（エンドツーエンド）テストです。この章では、主に単体テストと、Componentの結合テストを扱います。

Angularでは、テストの実行環境（テストランナー）として、長らくKarmaとJasmineが使われてきました。しかし、Angular 22では、Vitestが新規プロジェクトの既定になりました。Vitestは、高速で、設定も簡潔なモダンなテストランナーです。新規プロジェクトを`ng new`で作ると、Vitestとその実行に必要なものが用意されます。テストは、第3章で学んだCLIの`ng test`で実行します。

```bash
ng test
```

このコマンドで、アプリをテスト用にビルドし、Vitestがテストを実行します。既存のKarmaベースのプロジェクトも動きますが、新しく始めるならVitestが標準です。テストの書き方（`describe`や`it`、`expect`といった記法）は、Vitestでも従来とよく似ており、これまでの知識の多くがそのまま活きます。

## Serviceをテストする

もっとも書きやすいのが、Serviceのテストです。とくに、第22章で学んだ「状態を持たないService」は、入力に対する出力を確かめるだけなので、単純です。

```ts:src/app/pricing.spec.ts
import { PricingService } from './pricing';

describe('PricingService', () => {
  it('税込価格を計算する', () => {
    const service = new PricingService();
    expect(service.withTax(1000)).toBe(1100);
  });
});
```

`describe`でテストのまとまりを、`it`で個々のテストを表します。`expect(...).toBe(...)`で、「この値は、これと等しいはずだ」という期待を書きます。実際の値が期待と違えば、テストは失敗します。このServiceは依存を持たないため、`new`して直接テストできます。

依存を持つServiceは、その依存を偽物に差し替えてテストします。第23章で学んだDIの利点、「差し替えが容易」が、ここで活きます。本物の通信Serviceの代わりに、決まったデータを返す偽物を注入すれば、通信なしでロジックを確かめられます。

## TestBedでComponentをテストする

Componentのテストは、Serviceより少し複雑です。Componentは、テンプレートやDIと結びついているため、それらを含めた環境を用意する必要があります。その環境を作るのが、TestBedです。

```ts:src/app/greeting.spec.ts
import { TestBed } from '@angular/core/testing';
import { Greeting } from './greeting';

describe('Greeting', () => {
  it('名前を表示する', () => {
    const fixture = TestBed.createComponent(Greeting);
    fixture.componentRef.setInput('name', '山田');
    fixture.detectChanges(); // 変更検知を走らせる

    const text = fixture.nativeElement.textContent;
    expect(text).toContain('山田');
  });
});
```

`TestBed.createComponent`で、テスト対象のComponentを生成します。返ってくる`fixture`（フィクスチャ）は、そのComponentと、そのDOMへのアクセスを束ねたものです。`setInput`で入力を与え、`detectChanges()`で変更検知を走らせて、表示を更新します。そのうえで、`nativeElement`からDOMのテキストを取り出し、期待どおりかを確かめます。「入力を与え、表示を確認する」という、第4部で学んだデータフローが、そのままテストの形になります。

## Component Harnessで安定させる

Componentのテストで、DOMを直接調べると、ひとつ問題があります。テンプレートの構造（HTMLの組み立て方やクラス名）に、テストが依存してしまうことです。見た目を少し変えただけで、動作は同じなのにテストが壊れる、ということが起こります。

この問題を解決するのが、Component Harness（コンポーネントハーネス）です。`@angular/cdk/testing`が提供する仕組みで、Componentを、その内部構造ではなく、「役割」を通して操作します。たとえば、ボタンのHarnessなら、「そのボタンをクリックする」という操作を、DOMの詳細を知らずに行えます。

Harnessを使うと、テンプレートの構造が変わっても、役割が同じならテストは壊れません。テストが、実装の細部ではなく、振る舞いに向くようになります。Angular Materialのような部品ライブラリは、Harnessを提供しており、それらを使ったテストを安定して書けます。自作のComponentにも、Harnessを用意できます。DOMを直接調べるテストより、Harnessを使うほうが、長期的に壊れにくいテストになります。

## 何をテストすべきか

テストは、書けば書くほどよい、というものではありません。すべてを網羅しようとすると、テストの保守自体が負担になります。優先すべきは、次のようなものです。

- **業務ロジック**: 価格計算や、複雑な条件判断など、間違えると影響が大きい部分。
- **状態の変化**: Store（第10部）の、Actionやメソッドによる状態変化。
- **重要な分岐**: エラー処理や、権限による表示の出し分けなど。

逆に、単純な表示だけのComponentや、フレームワークが保証している部分まで、細かくテストする必要は薄いものです。第22章で責務を分離し、業務ロジックをServiceに切り出しておくと、この「重要な部分」がテストしやすい形になります。テストのしやすさは、設計のよさと表裏一体なのです。

## Signalとテスト

Signalベースのコードは、テストがしやすいという特長もあります。Signalは、`()`で現在の値を同期的に読めるため、テストの中で状態を確認するのが簡単です。`computed()`による派生値も、依存するSignalをセットして、結果を読むだけで確かめられます。

```ts:src/app/cart.spec.ts
it('商品を追加すると合計が増える', () => {
  const store = new CartService();
  store.add({ id: '1', name: '本', price: 500, quantity: 1 });

  expect(store.totalPrice()).toBe(500);
});
```

`store.totalPrice()`を呼ぶだけで、派生した合計金額を確認できます。RxJSのObservableのように購読を管理する必要がなく、非同期の完了を待つ必要もありません。第6部で学んだSignalが、状態管理だけでなく、テスト容易性の面でも利点をもたらすのです。モダンAngularのSignalファーストな書き方は、テストのしやすさという形でも報われます。

## 依存を差し替えてテストする

依存を持つComponentやServiceのテストでは、その依存を偽物に差し替えます。第23章で学んだDIの「差し替えが容易」という利点が、ここで具体的に活きます。TestBedの`providers`で、本物の代わりに偽物を登録します。

```ts:src/app/user-list.spec.ts
it('ユーザー一覧を表示する', () => {
  // 本物のUserServiceの代わりに、決まったデータを返す偽物を登録
  const fakeService = { getUsers: () => [{ id: '1', name: '山田' }] };

  TestBed.configureTestingModule({
    providers: [{ provide: UserService, useValue: fakeService }],
  });

  const service = TestBed.inject(UserService);
  expect(service.getUsers().length).toBe(1);
});
```

第25章で学んだProviderの`useValue`が、テストで役立つ場面です。本物のServiceが通信をしたり、複雑な準備を要したりしても、偽物に差し替えれば、テスト対象のロジックだけに集中できます。こうした偽物は、テストダブルと呼ばれ、単に決まった値を返すもの、呼ばれたことを記録するものなど、目的に応じた種類があります。重要なのは、テスト対象の外側を偽物で固定し、対象そのものの振る舞いを、安定して確かめられるようにすることです。DIを前提に設計されたAngularは、この差し替えが自然に行えるよう作られています。

## よくあるつまずき

- **`detectChanges()`を忘れる**: Componentの入力を変えても、`fixture.detectChanges()`を呼ばないと、テンプレートに反映されません。表示を確認する前に、変更検知を走らせます。
- **DOMの構造に依存しすぎる**: 特定のクラス名やHTML構造を前提にしたテストは、見た目の変更で壊れます。Component Harnessや、役割に着目した検証で、壊れにくくします。
- **何でもテストしようとする**: 網羅率を追い求めると、テストの保守が負担になります。重要なロジックや分岐を優先し、単純な表示は軽くします。
- **実際の通信やタイマーを使う**: テストで本物の通信や時間待ちをすると、遅く不安定になります。次章で学ぶ差し替えや`fakeAsync`で、速く安定させます。

## まとめ

- Angular 22では、テストランナーの既定がVitestになりました
- Serviceのテストは、入力に対する出力を`expect`で確かめる、単純な形です
- Componentは、TestBedで環境を作り、入力を与えて表示を確認します
- Component Harnessを使うと、テンプレート構造に依存しない安定したテストが書けます
- 業務ロジックや状態変化など、重要な部分を優先してテストします

次章では、RxJSやNgRx、非同期処理を含む、より難しいテストの戦略を学びます。
