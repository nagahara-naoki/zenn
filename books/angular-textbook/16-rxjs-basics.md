---
title: "RxJSの基礎（Observable・購読・Operator）"
---

この章では、非同期処理を扱うRxJSの基礎を学びます。中心概念のObservable、その購読とライフサイクル、そして値を加工するOperatorを扱います。

:::message
**この章で学ぶこと**

- 同期処理と非同期処理の違い
- Observableとは何か
- Subscriptionと`unsubscribe`
- 購読を解除しないと起きる問題
- `pipe`によるOperatorの適用
- 値を変換・絞り込むOperator
:::

## Observableとリアクティブプログラミング

RxJSの学習は、Observable（オブザーバブル）という概念から始まります。Observableは、RxJSのすべての中心にある考え方です。ここをしっかり理解すれば、続くOperatorやSubjectも、その応用として捉えられます。逆に、ここがあいまいだと、RxJS全体がわかりにくく感じられます。この節では、時間をかけてObservableの本質をつかみます。

Observableを一言でいえば、「時間とともに値を流す、水道管のようなもの」です。ふつうの関数は、呼ぶと一度だけ値を返して終わります。しかしObservableは、時間の経過とともに、複数の値を、何度でも流せます。この「複数の値が、時間をかけて流れてくる」という性質を扱えることが、RxJSの強みです。まずは、なぜそんな仕組みが必要なのかから見ていきましょう。

### 同期と非同期

プログラムの処理には、大きく2種類あります。同期処理と非同期処理です。

同期処理は、書いた順に、その場で結果が得られる処理です。`1 + 2`を計算すれば、ただちに`3`が得られます。前の行が終わってから、次の行へ進みます。素直で、理解しやすい流れです。

一方、非同期処理は、結果がすぐには得られない処理です。サーバーへデータを要求しても、応答が返ってくるのは、しばらく後です。その間、プログラムは止まって待つわけにはいかないので、「結果が来たら、この処理をする」という約束だけをして、先へ進みます。タイマーやクリックのように、「いつ起きるかわからない出来事」も、非同期の一種です。

Webアプリケーションは、この非同期処理であふれています。通信、ユーザー操作、時間経過。これらをどう扱うかが、アプリづくりの大きなテーマです。RxJSは、この非同期処理を、統一された形で扱うために生まれました。

### Observableとは何か

Observableは、非同期に流れてくる値の連なりを表すオブジェクトです。「ストリーム（流れ）」とも呼ばれます。

たとえば、次のようなものは、すべてObservableで表せます。

- ボタンのクリック（クリックのたびに値が流れる）
- サーバーからの応答（応答が返ってきたときに値が流れる）
- 一定間隔のタイマー（一定時間ごとに値が流れる）
- 入力欄への文字入力（入力のたびに値が流れる）

これらに共通するのは、「値が、時間をかけて、複数回流れてくる（こともある）」という点です。Observableは、この共通の構造を、ひとつの型で表現します。クリックも通信もタイマーも、すべて「Observable」という同じ器に収まるため、同じ道具（後の章で学ぶOperator）で加工できるのです。

Observableは、3種類の通知を流します。

- **次の値（next）**: 流れてくる値そのもの
- **エラー（error）**: 途中で問題が起きたときの通知。これが流れると、そのObservableは終わる
- **完了（complete）**: もう値を流し終えたという通知。これが流れると、そのObservableは終わる

たとえば通信なら、応答というnextが流れて、completeで終わります。失敗すればerrorが流れます。この3種類の通知が、Observableの言語です。

### ObserverとSubscribe

Observableは、水道管にたとえられます。しかし、管を用意しただけでは、水は流れません。蛇口をひねって、はじめて水が出ます。この「蛇口をひねる」操作が、`subscribe`（購読）です。

Observableは、`subscribe`されるまで、何もしません。値を流し始めるのは、誰かが`subscribe`したときです。この性質は重要なので、覚えておいてください。「Observableを作っただけでは動かない。購読して初めて動く」のです。

`subscribe`するとき、流れてくる通知を受け取る側を、Observer（オブザーバー）と呼びます。Observerは、先ほどの3種類の通知に対応する処理を持ちます。

```ts
import { interval } from 'rxjs';

// 1秒ごとに数を流すObservable
const source$ = interval(1000);

// 購読して、流れてくる値を受け取る
source$.subscribe({
  next: (value) => console.log('値:', value),
  error: (err) => console.error('エラー:', err),
  complete: () => console.log('完了'),
});
```

`interval(1000)`は、1秒ごとに`0, 1, 2, ...`と数を流すObservableです。`subscribe`に渡したObserverの`next`が、値が流れるたびに呼ばれます。変数名の末尾に付けた`$`は、Observableであることを示す慣習です。必須ではありませんが、コードの読みやすさのために広く使われています。

### Observableを作る

Observableは、さまざまな方法で作れます。RxJSには、目的に応じた生成関数が用意されています。代表的なものを挙げます。

- **`of`**: 渡した値を、そのまま順に流して完了します。`of(1, 2, 3)`は`1, 2, 3`を流します。
- **`from`**: 配列やPromiseなどを、Observableに変換します。
- **`fromEvent`**: DOMイベントを、Observableにします。クリックの流れなどを作れます。
- **`interval`・`timer`**: 時間にもとづいて値を流します。

```ts
import { of, from, fromEvent } from 'rxjs';

of('a', 'b', 'c');              // a, b, c を流して完了
from([10, 20, 30]);             // 配列から 10, 20, 30 を流す
fromEvent(button, 'click');     // ボタンのクリックを流し続ける
```

実際のアプリでは、これらを直接使うことは、そう多くありません。HttpClientやRouterが返すObservableを受け取って使う場面がほとんどだからです。とはいえ、Observableが「どこから生まれるのか」を知っておくと、仕組みの理解が深まります。とくに`fromEvent`は、DOMイベントもObservableとして統一的に扱えることを、よく示しています。

### なぜPromiseではなくObservableか

非同期処理といえば、JavaScriptには`Promise`という標準の仕組みもあります。「なぜAngularは、`Promise`ではなくObservableを多用するのか」と疑問に思うかもしれません。両者には、重要な違いがあります。

`Promise`は、値を1回だけ返して終わります。一度解決したら、それきりです。一方、Observableは、値を何度でも流せます。1回で終わる通信にも使えますが、クリックの連続や、時間ごとの更新のように、複数回の値にも対応できます。この「複数回」を扱える点が、Observableの大きな強みです。

もうひとつの違いが、キャンセルです。`Promise`は、いったん始まると途中で止められません。Observableは、`unsubscribe`で途中でキャンセルできます。検索のような場面で古い通信を打ち切れるのは、この性質のおかげです（第39章で扱います）。さらに、`map`や`filter`といった豊富なOperatorで加工できる点も、`Promise`にはない利点です。1回きりの単純な非同期なら`Promise`でも十分ですが、Angularが扱う多様な非同期には、Observableのほうが適しているのです。

| 観点 | Promise | Observable |
|---|---|---|
| 値の回数 | 1回だけ | 何度でも |
| キャンセル | できない | `unsubscribe`で可能 |
| 加工 | 限定的 | 豊富なOperator |
| 実行の開始 | 作成時ただちに | 購読されたとき |

### リアクティブプログラミング

RxJSが体現しているのは、リアクティブプログラミングという考え方です。「値の流れ（ストリーム）を宣言的に組み立て、変化に反応させる」プログラミングのスタイルです。

従来の書き方では、「クリックされたら、この変数を更新して、この関数を呼んで……」と、起きたことに対する手続きを、ひとつずつ書いていました。リアクティブなスタイルでは、「クリックの流れを、こう加工して、こう表示する」と、流れの変換として宣言的に書きます。命令を並べるのではなく、データの流れとその変換を記述するのです。

この考え方は、第6部で学んだSignalとも通じます。Signalも、「値が変わったら、それに依存する計算が自動で反応する」という点で、リアクティブでした。RxJSとSignalは、どちらもリアクティブの仲間です。違いは、Signalが「現在の値」に主眼を置くのに対し、RxJSは「時間をかけて流れる値の連なり」と、その細かな時間的制御に強い点です。この2つの関係は、第41章で詳しく扱います。

### AngularのどこでRxJSに出会うか

RxJSは、Angularの随所で使われています。すでに本書でも、いくつか登場していました。

- **HttpClient**（第9部）: サーバーへの要求の結果は、Observableで返ってきます
- **Router**（第7部）: `ActivatedRoute`の`paramMap`は、Observableでした
- **Forms**（第9部）: フォームの値の変化は、`valueChanges`というObservableで受け取れます
- **`async`パイプ**（第16章）: Observableをテンプレートで購読する仕組みでした

つまり、Angularを使ううえで、RxJSは避けて通れません。これらの機能を深く使うには、Observableの理解が土台になります。次章以降で、その扱い方を具体的に学んでいきましょう。

### よくあるつまずき

- **Observableを作っただけで動くと思う**: Observableは、`subscribe`されるまで何もしません。「関数を定義しただけで、まだ呼んでいない」のと同じ状態です。値が流れないときは、購読しているかを確認します。
- **Observableを配列のように扱う**: Observableは、いま手元にある値の集まりではなく、これから流れてくる値の連なりです。`for`で回したり、要素数を数えたりはできません。値を得るには購読します。
- **RxJSを難しく考えすぎる**: 概念は独特ですが、実務で必要なのは限られた範囲です。「時間とともに流れる値を、購読して受け取る」という核だけ押さえれば、あとは必要に応じて広げられます。

## SubscriptionとObservableのライフサイクル

前節で、Observableは`subscribe`されて初めて動き出すことを学びました。この節では、その`subscribe`が返すSubscription（購読）と、購読を適切に終わらせるライフサイクル管理を扱います。地味なテーマに見えますが、ここを疎かにすると、メモリリークという厄介な問題を引き起こします。RxJSを安全に使うための、重要な回です。

購読には、始まりと終わりがあります。`subscribe`で始まり、`unsubscribe`で終わります。問題は、終わらせるのを忘れると、Observableが値を流し続け、それを受け取る処理も動き続けてしまうことです。とくに、破棄されたはずのComponentに紐づく購読が生き残ると、無駄な処理やメモリの浪費につながります。この節では、その仕組みと、モダンAngularでの安全な後始末の方法を学びます。

### Subscriptionと購読解除

`subscribe`は、Subscriptionというオブジェクトを返します。これは、その購読を表す「取っ手」のようなものです。この取っ手を使って、`unsubscribe`を呼ぶと、購読を終わらせられます。

```ts
import { interval } from 'rxjs';

const subscription = interval(1000).subscribe((value) => {
  console.log(value);
});

// 購読を終わらせる
subscription.unsubscribe();
```

`unsubscribe`を呼ぶと、Observableからの値の受け取りが止まります。`interval`は、放っておくと永遠に数を流し続けるObservableですが、`unsubscribe`すれば、そこで止まります。購読を始めたら、不要になったときに終わらせる。これが、購読管理の基本です。

### 購読を解除しないと何が起きるか

購読の解除を忘れると、メモリリークが起こりえます。メモリリークとは、不要になったはずのものがメモリ上に残り続け、少しずつ資源を圧迫していく問題です。

具体的に考えてみましょう。あるComponentが、`interval`を購読して、1秒ごとに何かを処理するとします。このComponentが画面から消え、破棄されたとします。しかし、購読を解除していないと、`interval`は値を流し続け、破棄されたComponentの処理が、なおも実行され続けます。Componentは見えなくなったのに、その一部が裏で動き続けているのです。

このようなComponentが、画面遷移のたびに増えていくと、動き続ける購読がどんどん積み重なります。やがて、アプリは重くなり、不可解な動作を示すようになります。第21章で`ngOnDestroy`による後始末の重要性に触れましたが、購読の解除は、その代表例です。

### Cold ObservableとHot Observable

購読を理解するうえで、ObservableにはColdとHotの2種類がある、という区別も押さえておきましょう。

Cold Observableは、購読されるたびに、新しく値の生成を始めます。購読者ごとに、独立した流れが作られます。たとえば、HTTP通信のObservableはColdで、購読するたびに新しい通信が発生します。2人が別々に購読すれば、通信も2回起きます。水道の比喩でいえば、購読者ごとに専用の管が引かれるイメージです。

Hot Observableは、購読者の有無にかかわらず値が流れており、複数の購読者がその同じ流れを共有します。たとえば、ボタンのクリックイベントはHotで、誰が購読していようといまいと、クリックは起きています。購読者は、途中から同じ流れに加わる形です。共有された1本の管を、複数人がのぞき込むイメージです。

この違いは、「購読するたびに処理が繰り返されるのか、共有されるのか」に影響します。意図せず通信が何度も走る、といった問題は、Cold Observableの性質を理解していないことが原因のことがあります。まずは「HTTPはCold（購読ごとに走る）」という代表例を押さえておけば十分です。

### 手動での購読解除は面倒

購読の解除が大切だとわかっても、すべてを手作業で管理するのは大変です。Componentがいくつもの購読を持つと、`ngOnDestroy`で、それらをひとつずつ`unsubscribe`することになります。

```ts:手動での購読解除（煩雑な例）
export class Dashboard implements OnDestroy {
  private sub1 = this.a$.subscribe(/* ... */);
  private sub2 = this.b$.subscribe(/* ... */);

  ngOnDestroy(): void {
    this.sub1.unsubscribe();
    this.sub2.unsubscribe();
  }
}
```

購読が増えるほど、この後始末は長くなり、書き忘れのリスクも高まります。そこで、モダンAngularには、これを自動化する、より良い方法が用意されています。

### takeUntilDestroyedによる自動解除

もっとも手軽なのが、`takeUntilDestroyed()`です。これはAngularがrxjs-interopとして提供する演算子で、Componentなどが破棄されるタイミングで、購読を自動的に解除します。Angular 16で導入されました。

```ts:src/app/dashboard.ts
import { Component, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ selector: 'app-dashboard', template: `...` })
export class Dashboard {
  private readonly service = inject(DataService);

  constructor() {
    this.service.data$
      .pipe(takeUntilDestroyed())
      .subscribe((data) => {
        // dataを使った処理
      });
  }
}
```

`pipe(takeUntilDestroyed())`を挟むだけで、このComponentが破棄されるときに、購読が自動で解除されます。`ngOnDestroy`を書く必要も、Subscriptionを保持する必要もありません。書き忘れの心配がなくなるのが、大きな利点です。コンストラクター内など、注入コンテキストで呼ぶのが基本です。

### asyncパイプがもっとも簡単

そもそも、可能な場面では、`subscribe`を自分で書かないのが、もっとも安全です。第16章で学んだ`async`パイプを使えば、購読も解除もAngularが引き受けてくれます。

```html
<!-- asyncパイプ：購読も解除も自動 -->
@if (data$ | async; as data) {
  <p>{{ data.title }}</p>
}
```

テンプレートで表示するだけなら、`async`パイプがもっとも簡潔で、後始末の心配もありません。手動の`subscribe`は、テンプレートに直接表示しない処理（副作用）が必要なときの手段、と考えるとよいでしょう。そして手動で購読するときは、`takeUntilDestroyed()`で解除を自動化する。この2つを使い分ければ、購読管理の悩みの大半は解消します。

### 完了時の後始末とfinalize

購読が終わるのは、`unsubscribe`のときだけではありません。Observableが完了（complete）したり、エラーで終わったりしたときも、購読は終わります。いずれの終わり方でも、共通の後始末をしたいことがあります。そのときに使えるのが、`finalize`というOperatorです。

```ts
import { finalize } from 'rxjs';

this.loading.set(true);
this.api.getData()
  .pipe(
    finalize(() => this.loading.set(false)), // 成功・失敗・解除いずれでも実行
    takeUntilDestroyed(),
  )
  .subscribe((data) => { /* ... */ });
```

`finalize`に渡した処理は、Observableがどう終わっても（完了・エラー・購読解除のいずれでも）実行されます。この例では、通信の成否にかかわらず、ローディング表示を確実に消しています。「終わったら必ずやること」を書く場所として、`finalize`は便利です。ローディングの解除は、その典型的な使い道です。

### 状態をSignalで持てば購読管理は減る

購読管理の話をしてきましたが、そもそも手動の購読を減らすことが、根本的な対策になります。第6部で学んだSignalや、それと連携する`toSignal()`（第41章で詳しく扱います）を使えば、購読と解除をAngularに任せられます。

つまり、購読管理の悩みの多くは、「手動の`subscribe`をできるだけ書かない」ことで避けられます。表示は`async`パイプかSignalに任せ、どうしても手動購読が必要なときだけ`takeUntilDestroyed()`を添える。この方針を徹底すれば、メモリリークの心配は、ほとんどしなくて済むようになります。RxJSの購読管理は、かつてAngular開発の悩みの種でしたが、モダンな道具立てによって、大きく軽減されているのです。

### よくあるつまずき

- **購読解除の書き忘れ**: 手動`subscribe`で解除を忘れると、メモリリークの原因になります。`takeUntilDestroyed()`や`async`パイプで自動化するのが安全です。
- **`subscribe`の中で`subscribe`する**: 購読の中でさらに購読すると（ネスト）、管理が難しくなります。次章のOperator（`switchMap`など）を使えば、平らに書けます。
- **何でも手動`subscribe`にする**: 表示目的なら`async`パイプで足ります。手動購読は、副作用が必要な場面に絞ります。
- **`takeUntilDestroyed()`を注入コンテキスト外で使う**: 引数なしで使う場合は、コンストラクターなどの注入コンテキストで呼ぶ必要があります。それ以外の場所で使うには、`DestroyRef`を渡します。

## RxJS Operatorsと非同期処理

前節までで、Observableと購読の基本を押さえました。この節では、RxJSの真価というべきOperator（オペレーター）を学びます。Operatorは、Observableが流す値を、加工したり、絞り込んだり、組み合わせたりする道具です。RxJSが強力なのは、この豊富なOperatorによって、複雑な非同期処理を宣言的に組み立てられるからです。

RxJSには100種類を超えるOperatorがありますが、そのすべてを覚える必要はありません。実務で頻出するのは、ごく一部です。この節では、よく使うOperatorに絞って、確実に理解することを目指します。とくに、非同期処理を扱ううえで欠かせない、高階Operator（`switchMap`など）は、丁寧に解説します。

### pipeでOperatorをつなぐ

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

### 値を変換・絞り込む基本Operator

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

### 入力を制御するOperator

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

### 非同期を合成する高階Operator

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

### エラーを扱うOperator

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

### 複数のObservableを組み合わせる

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

### 検索機能を組み立てる

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

### よくあるつまずき

- **高階Operatorの選び違い**: `switchMap`と`mergeMap`などを取り違えると、通信が打ち切られない、順序が乱れる、といった問題が起きます。「最新だけ欲しいのか、すべて処理したいのか」で選びます。
- **`subscribe`のネストで書く**: 購読の中で次の購読を始めるのではなく、`switchMap`などで平らにつなぎます。ネストは読みにくく、解除も難しくなります。
- **Operatorを覚えようとしすぎる**: 数が多いので圧倒されがちですが、実務で使うのは限られます。本節で挙げたものを起点に、必要になったとき調べれば十分です。

## まとめ

- 非同期処理は、結果がすぐには得られない処理で、Webアプリにあふれています
- Observableは、時間とともに流れてくる値の連なりを表すオブジェクトです
- Observableはnext・error・completeの3種類の通知を流します
- `subscribe`はSubscriptionを返し、`unsubscribe`で購読を終わらせます
- 購読解除を忘れると、破棄後も処理が動き続け、メモリリークを招きます
- Cold Observableは購読ごとに、Hot Observableは共有された流れとして値を流します
- Operatorは`pipe`に並べて使い、値の変換の流れを宣言的に組み立てます
- `map`・`filter`・`tap`・`take`が、変換と絞り込みの基本です
- `debounceTime`・`distinctUntilChanged`で、入力の流れを時間的に制御できます

次章では、値を流す側にもなれるSubjectと、Signalとの連携・実践的な組み合わせを学びます。
