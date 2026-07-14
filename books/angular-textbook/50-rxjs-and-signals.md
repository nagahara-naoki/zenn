---
title: "RxJSとSignalsの共存"
---

ここまで、RxJSのObservableと、第6部のSignalという、2つのリアクティブな仕組みを学んできました。どちらも「値の変化に反応する」という点で似ていますが、得意分野が異なります。モダンAngularでは、この2つを対立するものとしてどちらか一方を選ぶのではなく、それぞれの強みを活かして共存させます。

この章では、RxJSとSignalをつなぐ橋渡しの仕組みと、どちらをどの場面で使うべきかの指針を学びます。橋渡しには、Angularがrxjs-interopとして提供する`toSignal()`と`toObservable()`を使います。この2つを理解すれば、既存のRxJSベースのコードと、新しいSignalベースのコードを、無理なく組み合わせられるようになります。

:::message
**この章で学ぶこと**

- RxJSとSignalの得意分野の違い
- `toSignal()`でObservableをSignalに変換する
- `toObservable()`でSignalをObservableに変換する
- 両者の使い分けの指針
:::

## RxJSとSignalの得意分野

まず、2つの仕組みの違いを整理します。似ているようで、主眼が異なります。

Signalは、「現在の値」を扱うのが得意です。いつ読んでも、その時点の値が同期的に得られます。テンプレートとの相性がよく、変更検知とも直結しています（第29章）。状態を保持し、それをUIに反映する、という用途に向いています。読み取りが同期的で、購読の解除も要らず、扱いが単純です。

RxJSは、「時間をかけて流れる値」を扱うのが得意です。値がいつ、どの順で流れるかという時間的な側面を、細かく制御できます。`debounceTime`で待ち、`switchMap`で切り替え、`catchError`で回復する、といった非同期の複雑な流れは、RxJSの独壇場です。その代わり、購読の管理が必要で、現在の値を同期的に取り出すのは苦手です。

| 観点 | Signal | RxJS（Observable） |
|---|---|---|
| 主眼 | 現在の値 | 時間をかけた値の流れ |
| 読み取り | 同期的（`()`で即座に） | 購読が必要 |
| 得意 | 状態の保持・UI反映 | 非同期の合成・時間制御 |
| 後始末 | 不要 | 購読解除が必要 |

この違いから、「状態はSignal、非同期の流れはRxJS」という役割分担が自然に導かれます。そして、その境界をまたぐための橋が、これから学ぶ変換関数です。

## toSignalでObservableをSignalに変換する

`toSignal()`は、ObservableをSignalに変換します。RxJSで組み立てた非同期の流れの結果を、Signalとして受け取り、テンプレートで扱いやすくする、という使い方が典型です。Angular 16で導入され、v17で安定版になりました。

```ts:src/app/clock.ts
import { Component } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { interval, map } from 'rxjs';

@Component({
  selector: 'app-clock',
  template: `<p>経過秒数: {{ seconds() }}</p>`,
})
export class Clock {
  // Observableを、初期値0のSignalに変換
  protected readonly seconds = toSignal(interval(1000), { initialValue: 0 });
}
```

`toSignal(interval(1000))`で、1秒ごとに値を流すObservableを、Signalに変えています。テンプレートでは`seconds()`と、ふつうのSignalとして読めます。`toSignal()`の利点は2つあります。ひとつは、テンプレートで`async`パイプを使わずに、Signalとして直接読めること。もうひとつは、購読の解除を自動でやってくれることです。Componentが破棄されると、内部の購読も解除されます。`initialValue`は、Observableが最初の値を流す前の値を指定するオプションです。

HTTP通信の結果を`toSignal()`でSignalにすれば、RxJSベースの通信と、Signalベースの画面を、きれいにつなげます。第9部のHTTP通信でも、この組み合わせが登場します。

`toSignal()`には、いくつかのオプションがあります。`initialValue`のほかに、`requireSync`は、`BehaviorSubject`のように購読と同時に値を流すObservableに対して、初期値なしでも同期的に値を得られるようにするものです。同期的に必ず値があるとわかっているなら、これを使うと`undefined`を扱わずに済みます。これらのオプションは、変換元のObservableの性質に応じて選びます。多くの場合は`initialValue`を指定しておけば十分です。

## toObservableでSignalをObservableに変換する

逆方向の変換が、`toObservable()`です。SignalをObservableに変換します。Signalで持っている状態を、RxJSの強力なOperatorで加工したいときに使います。

```ts:src/app/search.ts
import { Component, signal } from '@angular/core';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs';

@Component({ selector: 'app-search', template: `...` })
export class Search {
  protected readonly query = signal('');

  // Signalを Observable にして、RxJSで加工し、再びSignalに戻す
  private readonly query$ = toObservable(this.query);
  protected readonly results = toSignal(
    this.query$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap((q) => this.api.search(q)),
    ),
    { initialValue: [] },
  );
}
```

この例は、検索の実装です。検索語を`query`というSignalで持ち、`toObservable()`でObservableに変換します。そこに、前章で学んだ`debounceTime`・`distinctUntilChanged`・`switchMap`を適用して、無駄のない検索の流れを作ります。最後に`toSignal()`で結果を再びSignalに戻し、テンプレートで表示します。

ここに、共存の理想的な形が表れています。入力と結果という「状態」はSignalで持ち、その間の「時間制御と非同期合成」はRxJSに任せる。両者の境界を`toObservable()`と`toSignal()`が橋渡ししています。Signalの手軽さと、RxJSの表現力を、同時に得られるのです。

## どちらを使うべきか

では、実際の開発で、どちらを選べばよいのでしょうか。判断の指針を示します。

- **状態を持ち、UIに表示する**: Signalを使います。Componentの状態、Serviceが保持する状態は、Signalが第一の選択肢です。
- **単純な非同期（1回の通信など）**: Signalへの変換（`toSignal()`）や、`async`パイプで素直に扱えます。
- **複雑な非同期（入力制御・連鎖・キャンセル）**: RxJSを使います。`debounceTime`や`switchMap`が必要な場面は、RxJSの出番です。
- **既存のRxJSベースのAPI**: HttpClientやForms、Routerが返すObservableは、そのまま使うか、`toSignal()`でSignalに変換して扱います。

本書が推奨するのは、「状態管理の中心はSignalに寄せ、RxJSは非同期の流れの制御に用いる」という方針です。かつてはRxJSが状態管理まで広く担っていましたが、Signalの登場で、その役割分担が整理されました。RxJSを捨てるのではなく、その得意分野に集中させる、と考えてください。

## RxJSはこれからも必要か

Signalが状態管理の主役になると、「RxJSはもう不要では」と思うかもしれません。しかし、そうではありません。RxJSは、Signalには置き換えられない領域を持っています。

複数の非同期処理を合成する、値の流れを時間的に制御する、キャンセルや再試行を扱う。こうした「イベントの流れ」を宣言的に組み立てる力は、依然としてRxJSにしかありません。また、HttpClientやRouter、Formsといったフレームワークの機能が、Observableを返す以上、RxJSの理解は欠かせません。Signalとの共存を前提に、RxJSは今後もAngular開発の重要な柱であり続けます。第2章で述べた「新旧を対立させない」姿勢は、ここでも当てはまります。新しいSignalと既存のRxJSは、競合ではなく分担の関係にあるのです。

## よくあるつまずき

- **何でもRxJSで書こうとする**: 単純な状態の保持までObservableで書くと、購読管理が増え、コードが複雑になります。状態はSignalに寄せ、RxJSは非同期の流れに絞ります。
- **何でもSignalで書こうとする**: 逆に、`debounceTime`や`switchMap`が必要な処理を、Signalと`effect()`で無理に書こうとすると、かえって煩雑になります。時間的な制御は、素直にRxJSを使います。
- **`toSignal()`の初期値を忘れる**: 非同期のObservableを`toSignal()`する際、`initialValue`を指定しないと、最初の値が来るまで`undefined`になります。テンプレート側でその状態を考慮するか、初期値を与えます。
- **変換を過剰に往復させる**: `toObservable()`と`toSignal()`を無闇に何度も往復させると、流れが追いにくくなります。境界は必要な箇所に絞り、変換の回数を最小限にします。

## まとめ

- Signalは現在の値の保持に、RxJSは時間をかけた非同期の流れに強みがあります
- `toSignal()`はObservableをSignalに変換し、購読解除も自動化します
- `toObservable()`はSignalをObservableに変換し、RxJSのOperatorで加工できます
- 状態はSignal、複雑な非同期はRxJS、という役割分担が基本です
- RxJSはSignal時代も、非同期の合成やフレームワーク連携で必要であり続けます

次章では、これまで学んだRouter・RxJS・Signal・状態管理を組み合わせた、実践的な設計を扱います。
