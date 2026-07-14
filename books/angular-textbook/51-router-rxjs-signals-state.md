---
title: "RouterとRxJS・Signals・状態管理"
---

第8部の締めくくりとして、これまで学んできた要素を組み合わせた、実践的な設計を扱います。Router（第7部）、RxJS（第8部）、Signals（第6部）、そしてService（第5部）。これらは、実際のアプリケーションでは単独ではなく、連携して働きます。この章では、それらがどう噛み合うのかを、具体的な場面を通して確認します。

とくに焦点を当てるのは、「URLの変化に応じてデータを取得し、画面に反映する」という、どのアプリにも現れる基本的な流れです。ここには、Routerのパラメーター、RxJSの非同期合成、Signalによる状態保持が、すべて登場します。この一連の流れを設計できるようになることが、この部の到達点です。次の第10部で状態管理を本格的に学ぶ前の、橋渡しの回でもあります。

:::message
**この章で学ぶこと**

- URLの変化に応じたデータ取得の流れ
- RouterのパラメーターとRxJSの組み合わせ
- Signalによる画面状態の保持
- 各要素の役割分担
:::

## 典型的な流れ — URLからデータへ

多くのアプリに共通する流れを考えます。「商品詳細ページ（`/products/:id`）を開くと、その商品のデータを取得して表示する」というものです。この流れには、いくつもの要素が関わります。

1. URLの`:id`が決まる、または変わる（Router）
2. その`id`をもとに、サーバーへデータを要求する（RxJS・HttpClient）
3. 取得したデータを、画面に表示する（Signal）

第33章で、`withComponentInputBinding()`により、`:id`が`input()`に結びつくことを学びました。ここに、RxJSとSignalを組み合わせると、URLの変化に追従してデータを取得する流れを、宣言的に書けます。

## Signal入力を起点に組み立てる

もっともモダンな書き方は、ルートパラメーターの`input()`（Signal）を起点にすることです。`id`が変わったら、それに応じてデータを取得し直したい。この「変化に応じて」を、`toObservable()`と`switchMap`で表現します。

```ts:src/app/product-detail.ts
import { Component, inject, input } from '@angular/core';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { switchMap } from 'rxjs';
import { ProductService } from './product';

@Component({
  selector: 'app-product-detail',
  template: `
    @if (product(); as p) {
      <h1>{{ p.name }}</h1>
    } @else {
      <p>読み込み中</p>
    }
  `,
})
export class ProductDetail {
  private readonly service = inject(ProductService);

  readonly id = input.required<string>(); // URLの :id が結びつく

  protected readonly product = toSignal(
    toObservable(this.id).pipe(
      switchMap((id) => this.service.findById(id)),
    ),
  );
}
```

流れを追ってみましょう。URLが`/products/42`なら、`id`は`'42'`になります。`toObservable(this.id)`が、この`id`の変化をObservableにします。`switchMap`が、`id`ごとにデータ取得を行い、`id`が変われば前の取得を打ち切って新しい取得に切り替えます。最後に`toSignal()`が、結果を`product`というSignalに変え、テンプレートで表示します。

`/products/42`から`/products/99`へ移ったときも、この流れが自動で働きます。`id`が変わり、`switchMap`が古い通信を打ち切って新しい通信を始め、`product`が更新されます。Router・RxJS・Signalが、それぞれの役割を果たしながら、ひとつの流れとして噛み合っているのがわかります。

## 各要素の役割分担

この設計を、役割の視点で整理します。ひとつの流れの中で、各技術が担っている部分が明確に分かれています。

- **Router**: URLと`:id`を管理し、`withComponentInputBinding()`で`input()`に橋渡しする
- **Signal入力**: URLの現在の`id`を、Componentの状態として保持する
- **RxJS**: `id`の変化をきっかけに、非同期のデータ取得を制御する（`switchMap`による切り替え）
- **Signal（結果）**: 取得したデータを、テンプレートで扱いやすい形で保持する

第41章で述べた「状態はSignal、非同期はRxJS」という分担が、ここでも貫かれています。URLという入力も、データという結果も、Signalとして扱い、その間の非同期の制御だけをRxJSに任せています。この一貫した方針が、コードの見通しを保ちます。

## Routerのイベントを扱う

Routerは、パラメーター以外にも、遷移に関するさまざまな情報をObservableで提供します。`Router`の`events`は、遷移の開始や完了といった出来事を流すObservableです。これを購読すると、「ページ遷移のたびに何かをする」といった処理が書けます。

```ts:src/app/analytics.ts
import { inject } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

export class Analytics {
  private readonly router = inject(Router);

  constructor() {
    this.router.events
      .pipe(
        filter((e) => e instanceof NavigationEnd), // 遷移完了だけに絞る
        takeUntilDestroyed(),
      )
      .subscribe(() => {
        // ページ遷移が完了するたびの処理（アクセス記録など）
      });
  }
}
```

ここでも、これまで学んだ道具が総動員されています。`Router.events`というObservableを、`filter`で遷移完了（`NavigationEnd`）だけに絞り、`takeUntilDestroyed()`で購読解除を自動化しています。RxJSのOperatorとinteropが、Routerのイベント処理を簡潔にしているのです。

## 状態管理へのつながり

この章で見たのは、ひとつのComponentの中で完結する範囲の連携でした。しかし、アプリが大きくなると、状態は複数のComponentにまたがり、より本格的な管理が必要になります。「取得した商品データを、詳細ページとカートページで共有したい」といった要求です。

こうした、Componentを越えた状態の管理は、第10部の主題です。そこでは、この章で使ったSignalやRxJSを土台に、Store Serviceや、NgRxといった、より大規模な状態管理の仕組みを扱います。この章で学んだ「Router・RxJS・Signalの連携」は、その状態管理の基礎体力になります。個々の技術がどう噛み合うかを理解していれば、大規模な仕組みも、その延長として捉えられます。

## Serviceに状態を集約する

ひとつのComponentで完結しない例として、一覧ページと詳細ページで、選択中の商品を共有する場面を考えます。この場合、状態をComponentではなく、Serviceに置きます。第22章で学んだ「状態を持つService」を、Signalで実装します。

```ts:src/app/product-store.ts
@Injectable({ providedIn: 'root' })
export class ProductStore {
  private readonly selectedId = signal<string | null>(null);
  private readonly service = inject(ProductService);

  // 選択中の商品を、派生状態として公開する
  readonly selected = computed(() => {
    const id = this.selectedId();
    return id ? this.service.findByIdSync(id) : null;
  });

  select(id: string): void {
    this.selectedId.set(id);
  }
}
```

一覧ページで`select`を呼べば、詳細ページの`selected`も自動で更新されます。両ページが同じ`ProductStore`のインスタンス（`providedIn: 'root'`により単一）を共有しているためです。ここでも、状態はSignal、というモダンAngularの方針が貫かれています。かつてBehaviorSubjectで書いていたこうしたStore Serviceが、Signalによってより簡潔に書けるようになりました。この発想を発展させたものが、第10部で学ぶ状態管理です。

## よくあるつまずき

- **Component間の共有を`input`／`output`で無理につなぐ**: 離れたページ間の状態共有を、バケツリレーで実現しようとすると破綻します。共有したい状態は、Serviceに置くのが基本です。
- **URLに表すべき状態をComponentに閉じ込める**: 「どの商品を見ているか」のような、共有・復元したい状態は、Componentの内部ではなくURLで表すと、ブックマークや戻る操作と自然に噛み合います。
- **非同期の合成を手続き的に書く**: URLの変化に応じた取得を、`subscribe`のネストで書くと追いにくくなります。`toObservable()`と`switchMap`で、宣言的な流れとして組み立てます。

## この部のまとめとしての位置づけ

この章は、第6部から第8部までの集大成にあたります。Signal（状態）、RxJS（非同期）、Router（URL）という3つの柱が、実際のアプリケーションではひとつの流れとして協調することを見てきました。それぞれを個別に学んだときには見えなかった、技術どうしのつながりが、具体的なコードを通して立ち上がってきたはずです。この「組み合わせて設計する」感覚こそ、中級から上級へ進むうえで欠かせないものです。

## まとめ

- 「URLの変化に応じてデータを取得し表示する」流れは、多くのアプリに共通します
- ルートパラメーターの`input()`を起点に、`toObservable()`と`switchMap`で組み立てられます
- Routerが橋渡し、RxJSが非同期制御、Signalが状態保持、と役割が分かれます
- `Router.events`を`filter`と`takeUntilDestroyed()`で扱い、遷移時の処理を書けます
- Componentを越えた状態の管理は、これらを土台に第10部で本格的に扱います

以上で第8部は終わりです。次の第9部では、ユーザーからの入力を扱うFormsと、サーバーとの通信を担うHTTP通信を学びます。
