---
title: "エラー・再試行・処理の終了"
---

ここからは、本書の締めくくりです。ストリームを壊れにくくするための仕組みを扱います。

この章のテーマは、3つです。エラーが起きたときにどう対処するか、失敗した処理をどう再試行するか、そして不要になった処理をどうキャンセルするか。どれも、実務でRxJSを使ううえで、避けて通れないテーマです。ここまでは「うまくいく前提」の話が多かったのですが、現実の通信は失敗しますし、ユーザーは途中で操作をやめます。そうした「うまくいかないとき」にどう振る舞うかを、ここで設計します。

まず`catchError`による回復から始め、`retry`と`timeout`で失敗条件を制御し、最後に`takeUntil`や`fromFetch`による終了と`finalize`での後始末へ進みます。

## エラーから回復する

### エラーが起きると何が起こるか

まず、エラーが起きたとき、ストリームがどうなるかを、しっかり確認します。「Observableの仕組み」の章で見たとおり、ストリームは`error`で終わります。エラーが流れると、そのあとには`next`も`complete`も流れません。

```text
入力:  --1--2--X
出力:  --1--2--X   （Xのあとは何も流れない）
```

ここが重要です。エラーを何も処理しないと、ストリームはそこで完全に止まってしまいます。たとえば、ボタンのクリックを監視していたストリームなら、一度エラーが起きただけで、以降のクリックにいっさい反応しなくなります。ユーザーからは「急にボタンが効かなくなった」という、原因の見えない不具合に見えます。だからこそ、エラーへの対処が欠かせないのです。

### catchErrorで代替値を返す

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

### 別のObservableへ切り替える

`catchError`が返すのは、どんなObservableでも構いません。だから、代替値を1つ流すだけでなく、別の取得元に切り替える、といったこともできます。

```ts
import { catchError } from 'rxjs';

primaryApi$.pipe(
  catchError(() => backupApi$), // 主が失敗したら予備へ
);
```

主のAPIが失敗したら、予備のAPIへ切り替える、というフォールバックが書けます。

### エラーを握りつぶさない

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

### catchErrorを置く位置

`catchError`は、置く位置で効果が大きく変わります。これは、Flattening Operatorと組み合わせるときに、とくに重要になります。初学者が事故を起こしやすいポイントなので、しっかり見ておきましょう。

検索の例で考えます。`switchMap`の「外側」に`catchError`を置くと、1回の検索が失敗しただけで、検索ストリーム全体が止まってしまいます。以降の入力に、反応しなくなります。

```ts
// 良くない例: 1回の失敗で検索全体が止まる
keyword$.pipe(
  switchMap((kw) => searchApi(kw)),
  catchError(() => of([])), // ここだと検索ストリームごと終わる
);
```

Innerで起きたエラーは`switchMap`の出力へ伝わります。外側に置いた`catchError`は、その出力を`of([])`へ差し替えて完了させるため、以降の`keyword$`の値は購読されません。

### InnerのエラーでOuterを止めない

止めないためには、`catchError`を、Inner Observableの「内側」に置きます。こうすると、エラーはInnerの中で処理され、Outerの検索ストリームは生き続けます。

**インクリメンタル検索 3/4: 1回の失敗から回復する**

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

## 再試行と待ち時間を設計する

### retryで再試行する

一時的な失敗なら、もう一度試せば成功することがあります。ネットワークが一瞬乱れただけ、というような場合です。`retry`は、エラーが起きたときに、購読をやり直すOperatorです。

```ts
import { retry } from 'rxjs';

request$.pipe(retry(3)); // 初回に加えて最大3回再試行（最大4試行）
```

回数を渡すと、その回数だけ再試行します。`retry(3)`は「合計3試行」ではなく、初回の購読と最大3回の再購読、つまり最大4試行です。3回再試行しても失敗したら、エラーを下流へ流します。

### 再試行の間隔と条件

すぐに再試行しても、また失敗することがあります。相手のサーバーが混んでいるときなどです。間隔を空けて再試行するには、設定オブジェクトを渡します。

```ts
import { retry, timer } from 'rxjs';

request$.pipe(
  retry({
    count: 3,
    delay: (_error, retryCount) => timer(retryCount * 1000),
  }),
);
```

`delay`に、Observableを返す関数を渡すと、そのObservableが値を流すまで待ってから、再試行します。ここでは、再試行の回数が増えるほど、待ち時間を長くしています（1回目は1秒、2回目は2秒…）。

### Exponential Backoffの考え方

再試行のたびに、待ち時間を倍にしていく方法を、Exponential Backoff（指数バックオフ）と呼びます。1秒、2秒、4秒、8秒、と間隔を広げていきます。

```ts
import { retry, timer } from 'rxjs';

request$.pipe(
  retry({
    count: 5,
    delay: (_error, retryCount) => {
      const baseDelay = 2 ** (retryCount - 1) * 1000;
      const jitter = Math.random() * 250;
      return timer(baseDelay + jitter);
    },
  }),
);
```

`retryCount`は1から始まるので、`retryCount - 1`を指数にすると、基本の待ち時間は1秒、2秒、4秒、8秒になります。さらに少量の乱数（jitter）を足すと、複数のクライアントが同時に再試行するのを避けやすくなります。間隔を広げるのは、弱っているサーバーへ再試行が集中し、復旧を妨げるのを防ぐためです。

### timeoutで応答待ちに上限を設ける

`timeout`は、一定時間のあいだ値が流れないと、エラーを流すOperatorです。応答が返らない通信を、いつまでも待たずに、打ち切れます。

```ts
import { timeout } from 'rxjs';

request$.pipe(timeout({ first: 5000 })); // 最初の応答まで最大5秒
```

数値だけの`timeout(5000)`は、最初の値だけでなく、その後の値どうしの間隔にも5秒の上限を設けます。1回だけ応答するHTTPを意図するなら、`first`と書くと目的が明確です。`timeout`が流すエラーは`catchError`で受け止め、利用者向けの表示などにつなげます。

### 再試行してよい失敗かを判断する

再試行では、「何を再試行するか」を見極めることが大切です。一時的なネットワーク障害、`429 Too Many Requests`、一部の`5xx`には再試行の余地があります。一方、入力不正や権限不足など、多くの`4xx`は同じ要求を繰り返しても成功しません。`Retry-After`が返る場合は、その指示も尊重します。

また、GETのような冪等な読み取りと違い、注文作成や決済などの書き込みを機械的に再試行すると、処理が重複するおそれがあります。サーバー側の冪等性キーなどがないかぎり、自動再試行しないのが安全です。再試行には必ず上限を設け、「再試行してよい処理を、上限付きで、間隔を空けて試す」形にします。

## 購読の終了と処理の中断を分ける

### takeUntilで購読を終える

最後に、キャンセルと後始末です。「購読解除とObservableの自作」の章で`unsubscribe`によるキャンセルを扱いましたが、Operatorでも同じように書けます。

`takeUntil`は、合図となるObservableが値を流したら、ストリームを完了させるOperatorです。「画面を閉じる」という合図で購読を終わらせる、といった使い方をします。

```ts
import { interval, takeUntil, fromEvent } from 'rxjs';

const stop$ = fromEvent(stopButton, 'click');

interval(1000)
  .pipe(takeUntil(stop$)) // stopButtonが押されたら完了
  .subscribe((value) => console.log(value));
```

`take`で件数を区切る場合や、`switchMap`が古いInnerを切り替える場合にも、上流の購読は解除されます。ただし、「通知を受け取らなくなること」と「Producerの処理そのものが止まること」は同じではありません。購読解除ではTeardown Logicが実行されますが、タイマーの停止や通信の中断まで行われるのは、sourceがその後始末を実装している場合だけです。

### fromFetchでHTTP通信を中断できるようにする

`from(fetch(...))`は、すでに始まったPromiseをObservableに変換するだけです。購読を解除すると結果は通知されなくなりますが、`fetch`自体は中断されません。購読解除と`AbortController`を連動させたいときは、`fromFetch`を使えます。

```ts
import { fromFetch } from 'rxjs/fetch';

const tasks$ = fromFetch('/api/tasks', {
  selector: (response) => {
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return response.json();
  },
});

const subscription = tasks$.subscribe((tasks) => renderList(tasks));
subscription.unsubscribe(); // 完了前ならAbortControllerを通じて中断する
```

`selector`の中にbodyの読み取りまで含めると、その処理が完了するまで中断可能な範囲に含められます。それでも、リクエストがサーバーへ届いたあとなら、クライアント側で中断してもサーバー側の処理まで取り消せるとは限りません。とくに書き込みでは、「unsubscribeしたから保存されていない」と判断しないでください。

### finalizeで終了処理を確実に実行する

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

## 回復方法を選べるか確認する

1. 1回の検索失敗後も、次の入力を受け付けたい場合、`catchError`はどこに置きますか。
2. `retry(3)`は、初回を含めて最大何回試行しますか。
3. `from(fetch(url))`の購読を解除すると、HTTP通信も必ず中断されますか。

:::details 解答
1. `switchMap`が返すInner Observableの中です。1回分の失敗だけを代替値へ変えます。
2. 最大4回です。初回の購読に加えて、最大3回再購読します。
3. 中断されません。通知は止まりますが、すでに始まったPromiseは止められません。中断が必要なら`fromFetch`や、`AbortController`をTeardown Logicへ組み込んだObservableを使います。
:::

## エラーの置き換え、再試行、後始末を分けて設計する

エラー処理・再試行・キャンセルの要点を整理します。

- エラーを処理しないと、ストリームはそこで完全に止まります。
- `catchError`は、エラーを代替値や別のObservableに差し替えます。対処できないエラーは`throwError`で再通知します。
- Outerを止めたくないときは、`catchError`をInner Observableの内側に置きます。
- `retry`の指定回数は再購読の回数です。間隔、上限、jitter、処理の冪等性まで設計します。
- 購読解除は通知を止めます。Producerの処理まで止まるかはTeardown Logicに依存し、HTTPには`fromFetch`を使えます。
- `takeUntil`・`take`で購読を終え、`finalize`でどの終了経路でも後始末を実行します。

次章では、本書の締めくくりとして、RxJSのテストとデバッグを扱います。`TestScheduler`による検証と、値の流れを追うデバッグの方法を見ます。
