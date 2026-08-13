---
title: "エラー処理・再試行・キャンセル"
---

ここからは、本書の締めくくりです。ストリームを壊れにくくするための仕組みを扱います。

この章のテーマは、3つです。エラーが起きたときにどう対処するか、失敗した処理をどう再試行するか、そして不要になった処理をどうキャンセルするか。どれも、実務でRxJSを使ううえで、避けて通れないテーマです。ここまでは「うまくいく前提」の話が多かったのですが、現実の通信は失敗しますし、ユーザーは途中で操作をやめます。そうした「うまくいかないとき」にどう振る舞うかを、ここで設計します。

まずエラー処理の`catchError`から始め、`retry`と`timeout`による再試行、最後に`takeUntil`や`finalize`によるキャンセルと後始末へと進みます。

## エラーが起きると何が起こるか

まず、エラーが起きたとき、ストリームがどうなるかを、しっかり確認します。「Observable・Observer・subscribeの仕組み」の章で見たとおり、ストリームは`error`で終わります。エラーが流れると、そのあとには`next`も`complete`も流れません。

```text
入力:  --1--2--X
出力:  --1--2--X   （Xのあとは何も流れない）
```

ここが重要です。エラーを何も処理しないと、ストリームはそこで完全に止まってしまいます。たとえば、ボタンのクリックを監視していたストリームなら、一度エラーが起きただけで、以降のクリックにいっさい反応しなくなります。ユーザーからは「急にボタンが効かなくなった」という、原因の見えない不具合に見えます。だからこそ、エラーへの対処が欠かせないのです。

## catchErrorで代替値を返す

`catchError`は、エラーを受け取って、別のObservableに差し替えるOperatorです。エラーの代わりに、代替の値を流せます。

```text
入力:  --1--2--X
           catchError(() => of(0))
出力:  --1--2--0|
```

```ts
import { of, map, catchError } from 'rxjs';

source$
  .pipe(
    map((n) => {
      if (n < 0) throw new Error('負の数です');
      return n;
    }),
    catchError((error) => {
      console.log('エラーを捕まえた:', error.message);
      return of(0); // 代替値を流す
    }),
  )
  .subscribe((value) => console.log(value));
```

`catchError`に渡した関数が返すObservableに、流れが差し替わります。ここでは`of(0)`を返しているので、エラーの代わりに`0`が流れて、完了します。エラーで止まる代わりに、「もしダメだったら、この値を使う」という保険をかけている、というわけです。

## 別のObservableへ切り替える

`catchError`が返すのは、どんなObservableでも構いません。だから、代替値を1つ流すだけでなく、別の取得元に切り替える、といったこともできます。

```ts
import { catchError } from 'rxjs';

primaryApi$.pipe(
  catchError(() => backupApi$), // 主が失敗したら予備へ
);
```

主のAPIが失敗したら、予備のAPIへ切り替える、というフォールバックが書けます。

## エラーを握りつぶさない

`catchError`で気をつけたいのが、エラーを握りつぶさないことです。エラーを捕まえて何もせず、静かに握りつぶしてしまうと、問題が起きても誰も気づけません。あとで「なぜかデータが表示されない」と悩む原因になります。

対処できないエラーは、`throwError`で再通知して、下流へ伝えます。

```ts
import { catchError, throwError, of } from 'rxjs';

source$.pipe(
  catchError((error) => {
    if (error.status === 404) {
      return of(null); // 404は「データなし」として扱う
    }
    return throwError(() => error); // それ以外は再通知する
  }),
);
```

対処できるエラー（ここでは404を「データなし」とみなす）だけを代替値に変え、対処できないエラーは再通知する。この使い分けをすると、想定内のエラーはうまく処理しつつ、想定外の問題は見逃さずに済みます。

## catchErrorを置く位置

`catchError`は、置く位置で効果が大きく変わります。これは、Flattening Operatorと組み合わせるときに、とくに重要になります。初学者が事故を起こしやすいポイントなので、しっかり見ておきましょう。

検索の例で考えます。`switchMap`の「外側」に`catchError`を置くと、1回の検索が失敗しただけで、検索ストリーム全体が止まってしまいます。以降の入力に、反応しなくなります。

```ts
// 良くない例: 1回の失敗で検索全体が止まる
keyword$.pipe(
  switchMap((kw) => searchApi(kw)),
  catchError(() => of([])), // ここだと検索ストリームごと終わる
);
```

なぜこうなるかというと、内側の検索で起きたエラーが、外側の`keyword$`のストリームまで伝わり、そこで`keyword$`ごと終わってしまうからです。

## InnerのエラーでOuterを止めない

止めないためには、`catchError`を、Inner Observableの「内側」に置きます。こうすると、エラーはInnerの中で処理され、Outerの検索ストリームは生き続けます。

```ts
import { of, switchMap, catchError } from 'rxjs';

keyword$.pipe(
  switchMap((kw) =>
    searchApi(kw).pipe(
      catchError(() => of([])), // 検索1回分の中でエラーを処理する
    ),
  ),
);
```

`catchError`を、`searchApi(kw)`のすぐ内側に置きました。こうすると、1回の検索が失敗しても、その場で空の結果を返すだけで、外側の`keyword$`には影響しません。だから、次の入力には、ちゃんと反応します。「Outerを止めたくないなら、`catchError`はInnerの中へ」。これは実務で頻出する定石なので、しっかり覚えてください。

## retryで再試行する

一時的な失敗なら、もう一度試せば成功することがあります。ネットワークが一瞬乱れただけ、というような場合です。`retry`は、エラーが起きたときに、購読をやり直すOperatorです。

```ts
import { retry } from 'rxjs';

request$.pipe(retry(3)); // 失敗したら最大3回まで再試行
```

回数を渡すと、その回数だけ再試行します。3回試してもだめなら、あきらめて、エラーを下流へ流します。

## 再試行の間隔と条件

すぐに再試行しても、また失敗することがあります。相手のサーバーが混んでいるときなどです。間隔を空けて再試行するには、設定オブジェクトを渡します。

```ts
import { retry, timer } from 'rxjs';

request$.pipe(
  retry({
    count: 3,
    delay: (error, retryCount) => timer(retryCount * 1000), // 回数に応じて待つ
  }),
);
```

`delay`に、Observableを返す関数を渡すと、そのObservableが値を流すまで待ってから、再試行します。ここでは、再試行の回数が増えるほど、待ち時間を長くしています（1回目は1秒、2回目は2秒…）。

## Exponential Backoffの考え方

再試行のたびに、待ち時間を倍にしていく方法を、Exponential Backoff（指数バックオフ）と呼びます。1秒、2秒、4秒、8秒、と間隔を広げていきます。

```ts
import { retry, timer } from 'rxjs';

request$.pipe(
  retry({
    count: 5,
    delay: (error, retryCount) => timer(2 ** retryCount * 1000),
  }),
);
```

なぜ間隔を広げるのでしょうか。一度に大量の再試行が集中すると、ただでさえ弱っているサーバーの負荷をさらに増やし、かえって復旧を妨げてしまうからです。間隔を少しずつ広げることで、負荷を抑えながら、粘り強く再試行できます。

## timeoutで処理を打ち切る

`timeout`は、一定時間のあいだ値が流れないと、エラーを流すOperatorです。応答が返らない通信を、いつまでも待たずに、打ち切れます。

```ts
import { timeout } from 'rxjs';

request$.pipe(
  timeout(5000), // 5秒以内に応答がなければエラー
);
```

`timeout`が流すエラーは、`catchError`で受け止めて、「時間内に応答がありませんでした」といったメッセージの表示につなげられます。

## 通信エラーと業務エラー、無限リトライを避ける

再試行では、「何を再試行するか」を見極めることが大切です。通信の一時的な失敗は、再試行する価値があります。しかし、入力値が不正だといった業務エラー（たとえばHTTPの400番台）は、何度試しても、結果は変わりません。同じ入力を送り続けるだけだからです。こうしたエラーは、再試行せずに、すぐ利用者へ伝えます。

もう1つ、再試行の回数には、必ず上限を設けてください。上限のない再試行は、失敗し続けるとサーバーを叩き続け、問題を広げてしまいます。「一時的な失敗を、上限付きで、間隔を空けて再試行する」。これを基本の形にしてください。

## takeUntil・take・finalizeでキャンセルと後始末

最後に、キャンセルと後始末です。「Subscription・購読解除・Observableの自作」の章で`unsubscribe`によるキャンセルを扱いましたが、Operatorでも同じように書けます。

`takeUntil`は、合図となるObservableが値を流したら、ストリームを完了させるOperatorです。「画面を閉じる」という合図で購読を終わらせる、といった使い方をします。

```ts
import { interval, takeUntil, fromEvent } from 'rxjs';

const stop$ = fromEvent(stopButton, 'click');

interval(1000)
  .pipe(takeUntil(stop$)) // stopButtonが押されたら完了
  .subscribe((value) => console.log(value));
```

`take`で件数を区切るのも、キャンセルの一種です。そして、`switchMap`による古い処理の解除も、「Flattening Operator」の章で見たとおり、キャンセルの働きをします。RxJSでは、こうした購読解除が、`fetch`の`AbortSignal`のように「処理そのものを止める」動きにつながります。購読が解除されると、Teardown Logicが走り、通信やタイマーが止まるのです。

## 終了処理を確実に実行する

どんな終わり方をしても、必ず実行したい後始末は、`finalize`に書きます。`finalize`は、`complete`・`error`・`unsubscribe`のいずれで終わっても、必ず呼ばれるOperatorです。

```ts
import { finalize } from 'rxjs';

request$.pipe(
  finalize(() => {
    loading = false; // 成功でも失敗でも中断でも、ローディングを解除する
  }),
);
```

ローディング表示の解除は、その典型例です。通信が成功しても、失敗しても、途中でキャンセルされても、ローディングは必ず解除したいはずです。`finalize`に書けば、`complete`と`error`の両方に同じ処理を書く必要がなくなり、書き忘れも防げます。終わり方によらず必ず行いたい処理は、`finalize`に置くと安全です。

## エラーの置き換え、再試行、後始末を分けて設計する

エラー処理・再試行・キャンセルの要点を整理します。

- エラーを処理しないと、ストリームはそこで完全に止まります。
- `catchError`は、エラーを代替値や別のObservableに差し替えます。対処できないエラーは`throwError`で再通知します。
- Outerを止めたくないときは、`catchError`をInner Observableの内側に置きます。
- `retry`は再試行し、間隔を空ける設定やExponential Backoffで負荷を抑えます。
- 通信エラーは再試行し、業務エラーは再試行しません。回数には必ず上限を設けます。
- `takeUntil`・`take`でキャンセルし、`finalize`でどう終わっても後始末を実行します。

次章では、本書の締めくくりとして、RxJSのテストとデバッグを扱います。`TestScheduler`による検証と、値の流れを追うデバッグの方法を見ます。
