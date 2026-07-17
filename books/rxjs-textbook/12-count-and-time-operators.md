---
title: "件数と時間を制御する"
---

前章では、値そのものを変換したり選別したりするOperatorを見ました。この章では、値を「いくつ受け取るか」「いつ受け取るか」を制御するOperatorを扱います。

前半は件数の制御です。`take`や`first`で、必要な数だけ受け取って、自動的にストリームを終わらせる方法を見ます。後半は時間の制御です。`debounceTime`や`throttleTime`で、短い間隔で押し寄せる大量の値を、うまく間引く方法を見ます。とくに検索フォームやスクロールのような、実務で頻出する場面で活躍するOperatorたちです。

時間を制御するOperatorは、Marble Diagramがあると理解が一気に進みます。図とコードを合わせて確認していきましょう。

## takeとskipで件数を絞る

最初の数個だけ受け取りたいときは、`take`を使います。指定した数の値を受け取ると、自動的に`complete`します。

```text
入力:  --1--2--3--4--5-->
           take(3)
出力:  --1--2--3|
```

逆に、最初の数個を読み飛ばしたいときは、`skip`です。指定した数の値を捨て、そのあとを流します。

```text
入力:  --1--2--3--4--5-->
           skip(2)
出力:  -------3--4--5-->
```

```ts
import { interval, take, skip } from 'rxjs';

interval(1000)
  .pipe(take(3))
  .subscribe((value) => console.log('take:', value));

// 出力:
// take: 0
// take: 1
// take: 2   （ここで自動的に完了）
```

終わらないストリームを扱うとき、`take`はとくに便利です。`interval`のように自分では終わらないストリームでも、`take`を付ければ、指定した回数で自動的に終わります。前章までは`unsubscribe`で手動で止めていましたが、`take`なら「3個受け取ったら終わり」と、宣言的に書けるのです。

## firstとlast

最初の値を1つだけ受け取りたいときは、`first`です。`first`は、最初の値を受け取ると`complete`します。条件を渡すと、条件に合う最初の値を受け取ります。いっぽう`last`は、`complete`したときの、最後の値を受け取ります。

```ts
import { of, first, last } from 'rxjs';

of(1, 2, 3, 4).pipe(first()).subscribe((v) => console.log('first:', v)); // 1
of(1, 2, 3, 4).pipe(last()).subscribe((v) => console.log('last:', v)); //  4

// 条件に合う最初の値
of(1, 2, 3, 4).pipe(first((n) => n > 2)).subscribe((v) => console.log(v)); // 3
```

## take(1)とfirst()の違い

`take(1)`と`first()`は、どちらも「最初の1つを受け取って終わる」ように見えます。ふだんは同じ動きをします。では、何が違うのでしょうか。違いが出るのは、値が1つも流れずにストリームが終わったとき、つまり空だったときです。

`take(1)`のほうは、値がなくても、何も流さずに静かに`complete`します。対して`first()`は、値がないと`EmptyError`というエラーを流します。

```ts
import { EMPTY, take, first } from 'rxjs';

EMPTY.pipe(take(1)).subscribe({
  complete: () => console.log('take: 完了（エラーなし）'),
});

EMPTY.pipe(first()).subscribe({
  error: (e) => console.log('first: エラー ->', e.name),
});

// 出力:
// take: 完了（エラーなし）
// first: エラー -> EmptyError
```

違いを表にまとめます。

| Operator | 値があるとき | 値がないとき |
|---|---|---|
| `take(1)` | 最初の1つを流して完了 | 何もせず完了 |
| `first()` | 最初の1つを流して完了 | `EmptyError`を流す |

使い分けの考え方はこうです。「必ず値があるはず」という前提を明示し、なければ異常として扱いたいなら`first()`。値がなくても構わないなら`take(1)`。`first()`が空でエラーになるのは、一見不便に思えますが、「値が来ないのはおかしい」という状況を早く気づけるという意味では、むしろ安全です。

## takeWhileで条件が続くあいだ受け取る

条件を満たすあいだ値を流したいときは、`takeWhile`を使います。条件を満たさなくなった時点で、`complete`します。

```text
入力:  --1--2--3--4--1-->
           takeWhile(n => n < 3)
出力:  --1--2|
```

`3`で条件（`n < 3`）を満たさなくなるので、そこで完了します。`filter`と似ていますが、違いがあります。`filter`は、条件に合わない値を飛ばして流れを続けます。`takeWhile`は、条件を満たさなくなった時点で、ストリームそのものを終わらせます。「条件を満たすあいだだけ受け取り、外れたらもう終わり」というときに使います。

## 自動的にcompleteさせる

ここまでの`take`、`first`、`takeWhile`には、共通の性質があります。どれも、条件を満たすと自動的に`complete`することです。

この性質は、購読解除の手間を減らしてくれます。終わらないストリームでも、これらのOperatorで終わらせれば、`unsubscribe`を明示的に書かなくても、ストリームがきちんと片付きます。ただし、条件が満たされないかぎり終わらないもの（`takeWhile`で条件がずっと真の場合など）もあるので、「本当に終わる保証があるか」は意識してください。

## 時間を制御するOperator

ここからは、時間を制御するOperatorに移ります。ユーザーの入力やスクロールのように、短い間隔で大量に発生する値を、そのまま1つずつ処理すると、負荷が高くなります。たとえば、1文字入力するたびにAPIを呼んでいたら、通信が大量に飛んでしまいます。時間を制御するOperatorは、こうした値をうまく間引いてくれます。

代表格が`debounceTime`と`throttleTime`です。この2つの違いをつかむのが、この節のゴールです。

## debounceTimeで入力が止まるまで待つ

`debounceTime`は、値が届いてから指定時間が経つまで待ち、そのあいだに次の値が来なければ、最後の値を流します。逆にいえば、次々に値が来ているあいだは、何も流しません。

```text
入力:  --a-b-c------d--e----->
           debounceTime(200ミリ秒相当)
出力:  --------c--------e---->
```

`a`、`b`、`c`と続けて入力されているあいだは、じっと待ちます。入力が止まった`c`のあとで、はじめて`c`を流します。つまり「入力が落ち着いてから処理する」という動きです。これは、検索フォームのインクリメンタル検索に、そのまま当てはまります。ユーザーがタイプしている最中は待ち、手が止まったら検索する、というわけです。

```ts
import { fromEvent, map, debounceTime, distinctUntilChanged } from 'rxjs';

const input = document.querySelector('input')!;

fromEvent<InputEvent>(input, 'input')
  .pipe(
    map((event) => (event.target as HTMLInputElement).value),
    debounceTime(300),
    distinctUntilChanged(),
  )
  .subscribe((keyword) => console.log('検索:', keyword));
```

`debounceTime(300)`で入力が止まるのを300ミリ秒待ち、`distinctUntilChanged`で「前回と同じキーワード」を除いています。前章で学んだ`distinctUntilChanged`が、ここで生きてきます。この2つの組み合わせは、インクリメンタル検索の定番です。

## throttleTimeで連続通知を抑える

いっぽう`throttleTime`は、値を1つ流したあと、指定時間のあいだ、次の値を無視します。時間が過ぎると、また次の値を流します。

```text
入力:  --a-b-c-d-e-f-->
           throttleTime(300ミリ秒相当)
出力:  --a-----d-----g->
```

最初の`a`を流し、しばらく無視し、時間が過ぎた`d`をまた流します。`debounceTime`が「止まるまで待つ」のに対し、`throttleTime`は「一定間隔で間引く」動きです。両者はよく対比されるので、違いをはっきりさせておきましょう。`debounceTime`は落ち着いてから1回、`throttleTime`は一定のペースで定期的に、です。スクロールイベントの監視や、ボタンの連打を抑えたいときには、`throttleTime`が向いています。

## auditTimeとsampleTime

`auditTime`と`sampleTime`も、時間で値を間引くOperatorです。`debounceTime`・`throttleTime`と合わせて、4つまとめて動きを整理しておきます。

| Operator | 動き |
|---|---|
| `debounceTime` | 値が止まってから、最後の値を流す |
| `throttleTime` | 値を流したら、一定時間は無視する |
| `auditTime` | 値が来たら一定時間待ち、その時点の最新を流す |
| `sampleTime` | 一定間隔ごとに、その時点の最新を流す |

まずは`debounceTime`と`throttleTime`の2つを押さえれば、多くの場面に対応できます。`auditTime`と`sampleTime`は、より細かい制御が必要になったときの選択肢として、頭の片隅に置いておけば十分です。

## delayで通知を遅らせる

すべての通知を、指定時間だけ遅らせるのが`delay`です。値を間引くのではなく、流れる時刻を、まとめて後ろへずらします。

```text
入力:  --1--2--3--|
           delay(200ミリ秒相当)
出力:  ----1--2--3--|
```

演出のために表示を少し遅らせたいときや、テストで通信の遅延を模したいときに使います。間引く系のOperatorとは目的が違う、という点に注意してください。

## 時間制御Operatorの使い分け

どれを選ぶかは、間引き方の違いで決めます。

- 入力が落ち着いてから1回だけ処理したい → `debounceTime`（検索フォーム）
- 短時間の連続を抑えて、一定間隔で処理したい → `throttleTime`（スクロール、連打防止）
- 一定間隔で最新の状態を見たい → `auditTime`・`sampleTime`
- 流れる時刻を遅らせたいだけ → `delay`

迷ったときは、まず「止まるのを待つのか、間隔で間引くのか」を考えると、候補が絞れます。インクリメンタル検索なら`debounceTime`、スクロール量の監視なら`throttleTime`、というように、扱う値の性質から選んでください。

## まとめ

この章では、件数と時間を制御するOperatorを確認しました。

- `take`・`skip`は件数で絞り、`take`は指定数で自動的に完了します。
- `first`・`last`は最初・最後の値を取ります。値がないとき`first()`はエラー、`take(1)`は静かに完了します。
- `takeWhile`は条件を満たすあいだ流し、満たさなくなると完了します。
- `debounceTime`は入力が止まるのを待ち、`throttleTime`は一定間隔で間引きます。
- `auditTime`・`sampleTime`は最新値を一定間隔で流し、`delay`は通知の時刻を遅らせます。
- 「止まるのを待つのか、間隔で間引くのか」を基準に選びます。

次章からは、複数のObservableを組み合わせる合成に入ります。まず、複数の最新値を組み合わせる`combineLatest`と`withLatestFrom`を扱います。
