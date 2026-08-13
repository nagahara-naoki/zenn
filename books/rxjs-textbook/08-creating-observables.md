---
title: "Observableを作る"
---

前章までで、Observableの仕組みと性質を確認しました。ここからは、Observableの作り方を、実践的に見ていきます。

RxJSには、よくある値の生成をカバーするために、Creation Functionが用意されています。毎回`new Observable`で自作しなくても、これらを使えば、たいていの生成は済みます。この章では、そのうちもっともよく使う5つ、`of`、`from`、`fromEvent`、`interval`、`timer`を扱います。

前章で`new Observable`による自作を体験しましたが、実際の開発では、こうしたCreation Functionを使うことがほとんどです。それぞれが何を作るのか、そしてどう使い分けるのかを、1つずつ押さえていきましょう。

## ofで値から作る

`of`は、渡した値を、そのまま順番に流すCreation Functionです。流し終えると`complete`します。

```ts
import { of } from 'rxjs';

of(10, 20, 30).subscribe({
  next: (value) => console.log(value),
  complete: () => console.log('完了'),
});

// 出力:
// 10
// 20
// 30
// 完了
```

数値でも文字列でもオブジェクトでも、渡したものが、そのまま順に流れます。使いどころは、決まった値をいくつか流したいときや、テストで固定の値を用意したいときです。もっとも基本的で、いちばん最初に覚えるCreation Functionです。

## fromで配列・Iterable・Promiseから作る

`from`を使うと、配列やPromiseなど、「複数の値を含むもの」や「あとで値が手に入るもの」を、展開して流せます。

配列を渡すと、その要素が1つずつ流れます。

```ts
import { from } from 'rxjs';

from([10, 20, 30]).subscribe((value) => console.log(value));

// 出力:
// 10
// 20
// 30
```

文字列もJavaScriptではIterable（順番に取り出せるもの）なので、1文字ずつ流れます。SetやMapのようなIterableも、同じように展開できます。

```ts
import { from } from 'rxjs';

from('abc').subscribe((value) => console.log(value));

// 出力:
// a
// b
// c
```

そして、`from`にPromiseを渡すと、解決された値が流れて`complete`します。これはとても便利です。`Promise`を返す既存の処理（`fetch`など）を、そのままストリームの世界に取り込めるからです。

```ts
import { from } from 'rxjs';

from(fetch('/api/tasks')).subscribe((response) => {
  console.log(response.status);
});
```

## ofとfromの違い

`of`と`from`は、名前も役割も似ているので、混同しやすいところです。とくに、配列を渡したときの動きが違うので、ここははっきり区別しておきましょう。つまずきやすいポイントです。

`of([10, 20, 30])`は、配列を「1つの値」として、そのまま流します。`from([10, 20, 30])`は、配列を「展開」して、要素を1つずつ流します。

```ts
import { of, from } from 'rxjs';

of([10, 20, 30]).subscribe((value) => console.log('of:', value));
from([10, 20, 30]).subscribe((value) => console.log('from:', value));

// 出力:
// of: [ 10, 20, 30 ]   ← 配列がまるごと、1回だけ流れる
// from: 10             ← 要素が1つずつ流れる
// from: 20
// from: 30
```

違いを表にまとめます。

| 渡すもの | `of` | `from` |
|---|---|---|
| `of(1, 2, 3)` / `from([1, 2, 3])` | 1、2、3を順に流す | 1、2、3を順に流す |
| 配列を1つ | 配列をそのまま1回流す | 要素を1つずつ流す |
| Promise | Promiseオブジェクトをそのまま流す | 解決値を流す |

覚え方はシンプルです。`of`は「渡したものを、そのまま」流します。`from`は「渡したものを、展開して」流します。配列やPromiseの中身を1つずつ取り出したいなら、`from`です。この使い分けは、実務でよく必要になります。

## fromEventでDOMイベントから作る

DOMイベントをObservableにするのが`fromEvent`です。前章で自作したものと同じ働きを、たった1行で書けます。

```ts
import { fromEvent } from 'rxjs';

const button = document.querySelector('button')!;
const clicks$ = fromEvent<MouseEvent>(button, 'click');

clicks$.subscribe(() => console.log('クリックされました'));
```

クリックだけではありません。入力、スクロール、キーボードなど、あらゆるDOMイベントを、同じように扱えます。

```ts
import { fromEvent } from 'rxjs';

const input = document.querySelector('input')!;

// 入力イベント
fromEvent<InputEvent>(input, 'input').subscribe((event) => {
  const value = (event.target as HTMLInputElement).value;
  console.log('入力:', value);
});

// キーボードイベント
fromEvent<KeyboardEvent>(document, 'keydown').subscribe((event) => {
  console.log('キー:', event.key);
});
```

`fromEvent`で作ったObservableは、前章で見たとおりHotです。イベントは、購読とは無関係に発生するからです。そして、うれしいことに、購読を解除すると、内部で登録したイベントリスナーが自動的に外れます。自分で`removeEventListener`を書く必要はありません。

```ts
const subscription = clicks$.subscribe(() => console.log('クリック'));

// 解除すると、addEventListenerで登録されたリスナーも自動で外れる
subscription.unsubscribe();
```

前章のTeardown Logicで学んだ「後片付け」を、`fromEvent`が肩代わりしてくれている、というわけです。

## intervalとtimer

`interval`と`timer`は、時間に沿って値を流すCreation Functionです。時間を扱う処理の基本になります。

`interval`は、指定した間隔ごとに、0から始まる数値を流し続けます。

```ts
import { interval } from 'rxjs';

interval(1000).subscribe((value) => console.log(value));

// 出力（1秒ごと）:
// 0
// 1
// 2
// ...
```

いっぽうの`timer`は、指定した時間だけ待ってから値を流します。第2引数を渡すと、そのあとは指定間隔で流し続けます。

```ts
import { timer } from 'rxjs';

// 3秒後に0を流して完了
timer(3000).subscribe((value) => console.log('timer:', value));

// 3秒後に0を流し、そのあと1秒ごとに1, 2, 3...
timer(3000, 1000).subscribe((value) => console.log('poll:', value));
```

2つの違いを整理します。

| Creation Function | 最初の通知 | その後 |
|---|---|---|
| `interval(1000)` | 1秒後に0 | 1秒ごとに1、2、3… |
| `timer(3000)` | 3秒後に0 | 完了する |
| `timer(3000, 1000)` | 3秒後に0 | 1秒ごとに1、2、3… |

使い分けの目安は、こうです。すぐに繰り返しを始めたいなら`interval`、最初だけ待ち時間を置きたいなら`timer`です。

## カウントダウンを作る

これらを応用して、カウントダウンを作ってみましょう。`interval`は0から増えていくので、それを`map`で「残り秒数」に変換します。

```ts
import { interval, map, take } from 'rxjs';

interval(1000)
  .pipe(
    map((count) => 5 - count),
    take(5),
  )
  .subscribe((remaining) => console.log(`残り${remaining}秒`));

// 出力:
// 残り5秒
// 残り4秒
// 残り3秒
// 残り2秒
// 残り1秒
```

`interval`が0, 1, 2...と流すのを、`map`で`5 - count`（つまり5, 4, 3...）に変換しています。ここで使った`take(5)`は、最初の5個を受け取ったら自動的に`complete`させるOperatorです。件数を制御するOperatorは「件数と時間を制御する」の章で詳しく扱うので、いまは「5個で止める指定」と考えてください。`interval`という材料が、別の値に姿を変えていく感覚をつかんでください。

## 定期ポーリングの起点にする

一定間隔でサーバーの状態を確認する処理を、ポーリングと呼びます。その起点として、`timer`や`interval`が使えます。

```ts
import { timer } from 'rxjs';

// すぐに1回、そのあと5秒ごとに合図を流す
const poll$ = timer(0, 5000);

poll$.subscribe(() => {
  console.log('ここでAPIを呼ぶ');
});
```

この例では、合図を流すところまでを示しました。実際に、その合図を受けてAPIを呼び、結果を受け取るには、「次の合図が来たとき、前の通信をどう扱うか」という判断が必要になります。その組み立ては、「Flattening Operator」の章で行います。ここでは、`timer`が定期的な合図の発生源になる、と押さえておいてください。

## 購読解除で停止する

`interval`や`timer`は、放っておくと動き続けます。前章までに見たとおり、必要がなくなったら`unsubscribe`で止めます。

```ts
import { interval } from 'rxjs';

const subscription = interval(1000).subscribe((value) => console.log(value));

setTimeout(() => subscription.unsubscribe(), 3500);
```

`unsubscribe`を呼ぶと、内部のタイマーが止まります。止めないと、画面から離れてもタイマーが動き続けてしまいます。

## タイマーの多重起動を防ぐ

タイマーで、とくに気をつけたいのが、多重起動です。前のタイマーを止めないまま新しいタイマーを始めると、複数のタイマーが同時に動いてしまう、という問題です。

たとえば、ボタンを押すたびにポーリングを開始するコードで、前の購読を解除し忘れると、押した回数だけタイマーが増えていきます。

```ts
import { interval } from 'rxjs';

let subscription;

button.addEventListener('click', () => {
  // 悪い例: 前のsubscriptionを解除していない
  subscription = interval(1000).subscribe((value) => console.log(value));
});
// ボタンを3回押すと、3つのintervalが同時に動いてしまう
```

防ぐには、新しく始める前に、前の購読を解除します。

```ts
import { interval, Subscription } from 'rxjs';

let subscription: Subscription | undefined;

button.addEventListener('click', () => {
  subscription?.unsubscribe(); // 前のタイマーを止めてから
  subscription = interval(1000).subscribe((value) => console.log(value));
});
```

このような「新しい処理を始めるとき、前の処理を止める」という動きは、実はRxJSではOperatorできれいに書けます。`switchMap`がその代表です。手作業で`unsubscribe`を管理するかわりに、Operatorで解決する方法を、「Flattening Operator」の章で見ます。いまは、多重起動という落とし穴があること、そして手作業での防ぎ方を、知っておいてください。

## 値の出どころに合わせてCreation Functionを選ぶ

よく使うCreation Functionの選び方を整理します。

- `of`は、渡した値をそのまま順番に流します。
- `from`は、配列・Iterable・Promiseを展開して流します。
- `of`と`from`は配列の扱いが違います。中身を1つずつ取り出したいなら`from`を使います。
- `fromEvent`はDOMイベントをObservableにし、購読解除でリスナーも自動で外れます。
- `interval`は一定間隔で、`timer`は待ち時間のあとに値を流します。
- タイマーは`unsubscribe`で止め、始める前に前の購読を解除して多重起動を防ぎます。

次章では、特殊な場面で使うCreation Functionと、PromiseとObservableを相互に変換する方法を扱います。
