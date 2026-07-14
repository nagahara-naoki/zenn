---
title: "RxJS Operatorsと非同期処理"
---

前章までで、Observableと購読の基本を押さえました。この章では、RxJSの真価というべきOperator（オペレーター）を学びます。Operatorは、Observableが流す値を、加工したり、絞り込んだり、組み合わせたりする道具です。RxJSが強力なのは、この豊富なOperatorによって、複雑な非同期処理を宣言的に組み立てられるからです。

RxJSには100種類を超えるOperatorがありますが、そのすべてを覚える必要はありません。実務で頻出するのは、ごく一部です。この章では、よく使うOperatorに絞って、確実に理解することを目指します。とくに、非同期処理を扱ううえで欠かせない、高階Operator（`switchMap`など）は、丁寧に解説します。

:::message
**この章で学ぶこと**

- `pipe`によるOperatorの適用
- 値を変換・絞り込むOperator
- 入力を制御するOperator
- 非同期を合成する高階Operator
:::

## pipeでOperatorをつなぐ

Operatorは、Observableの`pipe`メソッドに渡して使います。`pipe`は、複数のOperatorを順につなげ、値の変換の流れを作ります。

```ts
import { map, filter } from 'rxjs';

source$
  .pipe(
    filter((n) => n % 2 === 0), // 偶数だけ通す
    map((n) => n * 10),         // 10倍する
  )
  .subscribe((value) => console.log(value));
```

`pipe`の中にOperatorを並べると、値は上から順に、各Operatorを通っていきます。この例では、まず`filter`で偶数だけを残し、次に`map`で10倍しています。第16章のPipe（縦棒`|`）と考え方が似ていますが、こちらはObservableに対する変換です。加工の段階を、上から順に読めるのが特徴です。

## 値を変換・絞り込む基本Operator

まず、もっとも基本的なOperatorを押さえます。

- **`map`**: 各値を、別の値に変換します。配列の`map`と同じ発想です。
- **`filter`**: 条件に合う値だけを通します。合わない値は捨てられます。
- **`tap`**: 値を変えずに、途中で副作用（ログ出力など）を行います。デバッグに便利です。
- **`take`**: 先頭から指定した個数だけ受け取り、そこで完了します。

```ts
import { map, tap, take } from 'rxjs';

source$.pipe(
  tap((n) => console.log('変換前:', n)), // ログを出す（値は変えない）
  map((n) => n * 2),                     // 2倍する
  take(3),                               // 最初の3つだけ
);
```

これらは、配列を操作する感覚に近く、直感的に使えます。`map`で形を整え、`filter`で絞り、`tap`で様子を見る。この組み合わせが、変換の基本になります。

## 入力を制御するOperator

非同期、とくにユーザー入力を扱うとき、値の流れを時間的に制御したいことがあります。代表的なのが、次の2つです。

- **`debounceTime`**: 値が流れてから一定時間、次の値が来なければ、その値を通します。連続した入力を「落ち着くまで待つ」使い方をします。
- **`distinctUntilChanged`**: 直前と同じ値なら、通しません。値が実際に変わったときだけ流します。

これらは、検索ボックスの実装で威力を発揮します。1文字打つごとに検索を走らせると、通信が過剰になります。`debounceTime`で入力が落ち着くのを待ち、`distinctUntilChanged`で同じ語の再検索を防げば、無駄な通信を大きく減らせます。

```ts
import { debounceTime, distinctUntilChanged } from 'rxjs';

searchInput$.pipe(
  debounceTime(300),        // 入力が300ミリ秒落ち着くまで待つ
  distinctUntilChanged(),   // 前回と同じ語なら無視する
);
```

## 非同期を合成する高階Operator

RxJSでもっとも重要かつ、つまずきやすいのが、高階Operatorです。これは、「値が流れてきたら、別のObservableを始める」ためのOperatorです。たとえば「検索語が流れてきたら、その語でHTTP通信（これもObservable）を始める」といった、Observableの中でObservableを扱う場面で使います。

代表的な4つを、その性質とともに示します。

| Operator | 前の処理がまだ終わっていないとき |
|---|---|
| `switchMap` | 前の処理を打ち切り、新しい処理に切り替える |
| `mergeMap` | 前の処理と並行して、新しい処理も走らせる |
| `concatMap` | 前の処理の完了を待ってから、順番に処理する |
| `exhaustMap` | 前の処理が終わるまで、新しい要求を無視する |

もっとも使うのが`switchMap`です。検索を例にとると、新しい検索語が来たら、前の検索の結果はもう不要です。`switchMap`は、前の通信を打ち切って、新しい通信に切り替えてくれます。古い結果が後から届いて表示が乱れる、という問題を防げます。

```ts
import { debounceTime, switchMap } from 'rxjs';

const results$ = searchInput$.pipe(
  debounceTime(300),
  switchMap((keyword) => this.api.search(keyword)), // 語ごとに通信、前のは打ち切る
);
```

使い分けの目安は、こうです。検索や最新の状態取得のように「最新だけが欲しい」なら`switchMap`。すべての要求を確実に処理したいなら`concatMap`（順番に）か`mergeMap`（並行に）。二重送信を防ぎたい保存ボタンのような場面は`exhaustMap`。まずは`switchMap`を軸に覚え、必要に応じて使い分けます。

## エラーを扱うOperator

非同期処理では、エラーへの備えも欠かせません。`catchError`は、Observableの流れの中でエラーが起きたときに、それを捉えて対処するOperatorです。

```ts
import { catchError, of } from 'rxjs';

this.api.getUser(id).pipe(
  catchError((err) => {
    console.error(err);
    return of(null); // 代わりの値を流して、流れを続ける
  }),
);
```

エラーが起きると、そのObservableは通常そこで終わってしまいます。`catchError`で捉え、`of(null)`のような代わりの値を返せば、流れを止めずに、エラー時の振る舞いを定義できます。通信の失敗に備えて、この`catchError`をよく使います。エラー処理の詳しい設計は、第9部のHTTP通信でも扱います。

## 複数のObservableを組み合わせる

複数のObservableを、ひとつにまとめたい場面もあります。そのための結合Operatorも、いくつか押さえておきましょう。

- **`combineLatest`**: 複数のObservableの、それぞれの最新値を組み合わせます。どれかが新しい値を流すたびに、全員の最新値をまとめて流します。複数の条件から結果を導くときに使います。
- **`forkJoin`**: 複数のObservableがすべて完了したときに、それぞれの最後の値をまとめて流します。複数の通信を並行して行い、全部そろってから処理したいときに使います。
- **`merge`**: 複数のObservableの値を、区別せずにひとつの流れにまとめます。

```ts
import { combineLatest, forkJoin } from 'rxjs';

// 絞り込み条件と並び順、両方の最新値から一覧を作る
combineLatest([this.filter$, this.sort$]).pipe(
  switchMap(([filter, sort]) => this.api.list(filter, sort)),
);

// ユーザー情報と注文履歴を並行取得し、両方そろってから処理
forkJoin({ user: this.api.getUser(id), orders: this.api.getOrders(id) });
```

`combineLatest`は「最新の組み合わせ」、`forkJoin`は「全部そろうのを待つ」と覚えると使い分けやすくなります。複数の非同期を扱うとき、これらが役立ちます。

## 検索機能を組み立てる

学んだOperatorを組み合わせて、実用的な検索機能を作ってみます。入力を制御するOperatorと、高階Operator、エラー処理を、ひとつの流れにまとめます。

```ts:src/app/product-search.ts
import { debounceTime, distinctUntilChanged, switchMap, catchError, of } from 'rxjs';

// searchInput$：検索語が流れてくるObservable
const results$ = searchInput$.pipe(
  debounceTime(300),           // 入力が落ち着くまで待つ
  distinctUntilChanged(),      // 同じ語の再検索を防ぐ
  switchMap((keyword) =>       // 語ごとに通信、前のは打ち切る
    this.api.search(keyword).pipe(
      catchError(() => of([])), // 失敗時は空の結果にする
    ),
  ),
);
```

この短い流れの中に、非同期処理の勘所が詰まっています。無駄な通信を`debounceTime`と`distinctUntilChanged`で抑え、古い結果を`switchMap`で打ち切り、失敗を`catchError`で吸収する。手続き的に書けば何十行にもなる制御が、宣言的な数行で表現できています。これが、Operatorを組み合わせる力です。`catchError`を内側の`switchMap`の中に置いているのは、通信が失敗しても外側の流れ全体は止めないための工夫です。

## よくあるつまずき

- **高階Operatorの選び違い**: `switchMap`と`mergeMap`などを取り違えると、通信が打ち切られない、順序が乱れる、といった問題が起きます。「最新だけ欲しいのか、すべて処理したいのか」で選びます。
- **`subscribe`のネストで書く**: 購読の中で次の購読を始めるのではなく、`switchMap`などで平らにつなぎます。ネストは読みにくく、解除も難しくなります。
- **Operatorを覚えようとしすぎる**: 数が多いので圧倒されがちですが、実務で使うのは限られます。本章で挙げたものを起点に、必要になったとき調べれば十分です。

## まとめ

- Operatorは`pipe`に並べて使い、値の変換の流れを宣言的に組み立てます
- `map`・`filter`・`tap`・`take`が、変換と絞り込みの基本です
- `debounceTime`・`distinctUntilChanged`で、入力の流れを時間的に制御できます
- 高階Operator（`switchMap`など）は、値をきっかけに別のObservableを始めます
- `catchError`でエラーを捉え、流れを止めずに対処できます

次章では、自分から値を流すこともできるSubjectと、その仲間たちを学びます。
