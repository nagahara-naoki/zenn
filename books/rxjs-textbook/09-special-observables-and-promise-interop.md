---
title: "特殊なObservableとPromise相互変換"
---

前章では、よく使うCreation Functionを見ました。この章では、少し特殊な場面で役立つものを扱います。

具体的には、購読するまでObservableの生成そのものを遅らせる`defer`、条件によってObservableを切り替える`iif`、そして「値を流さない」「エラーだけを流す」といった、変わったObservableを紹介します。どれも出番は多くありませんが、知っておくと、いざというときに役立ちます。

後半では、PromiseとObservableの相互変換を扱います。`async` / `await`で書かれた既存のコードと、RxJSを組み合わせる場面で必要になる知識です。あわせて、変換すべき場面と、すべきでない場面を整理します。

## deferで購読時にObservableを作る

`defer`は、購読された瞬間に、はじめてObservableを作るCreation Functionです。渡した関数が、購読のたびに呼ばれます。

「なぜそんなものが必要なのか」と思うかもしれません。理由を、具体例で見てみましょう。ふつうにObservableを書くと、その中の値が、コードを書いた時点で決まってしまうことがあります。

```ts
import { of } from 'rxjs';

const time$ = of(Date.now()); // ここでDate.now()が1回だけ評価される

time$.subscribe((t) => console.log(t));
// しばらく後
time$.subscribe((t) => console.log(t));

// 2回とも同じ時刻が出る（作成時の値が固定されている）
```

`of(Date.now())`は、`of`を書いた瞬間に`Date.now()`が評価され、その値が固定されてしまいます。だから、いつ購読しても、同じ（作成時の）時刻が流れます。「購読するたびに、そのときの時刻がほしい」のに、そうならないのです。

`defer`を使うと、この問題が解けます。購読するたびに関数が呼ばれるので、そのつど新しい値になります。

```ts
import { defer, of } from 'rxjs';

const time$ = defer(() => of(Date.now())); // 購読のたびにDate.now()を評価

time$.subscribe((t) => console.log(t));
// しばらく後
time$.subscribe((t) => console.log(t));

// 2回で違う時刻が出る（購読時に評価される）
```

## 作成時と購読時の違い

この「作成時」と「購読時」の違いは、購読の瞬間に最新の状態を読みたいときに、効いてきます。

たとえば、APIを呼ぶObservableを組み立てるとき、`from(fetch(...))`とそのまま書くと、前章までに見たとおり、`fetch`がその場で始まってしまいます。`fetch`は作った時点で動くからです。`defer`で包めば、購読するまで`fetch`を遅らせられます。

```ts
import { defer, from } from 'rxjs';

// 購読するまでfetchは実行されない
const tasks$ = defer(() => from(fetch('/api/tasks')));
```

要点はこうです。作成時に値や処理を確定させたくないとき、`defer`で購読時まで遅らせる。この使い分けを覚えておくと、意図しない先走りを防げます。

## iifで条件によってObservableを切り替える

`iif`は、条件によって2つのObservableを切り替えるCreation Functionです。購読した時点で条件を評価し、どちらのObservableを使うかを決めます。

```ts
import { iif, of, EMPTY } from 'rxjs';

function getValue(isLoggedIn: boolean) {
  return iif(
    () => isLoggedIn,
    of('ようこそ'), // 条件がtrueのとき
    EMPTY, //          条件がfalseのとき
  );
}

getValue(true).subscribe((v) => console.log(v)); // ようこそ
getValue(false).subscribe({ complete: () => console.log('何も流れず完了') });
```

条件を「購読時に」評価したいときに、`iif`が向いています。単純な分岐なら、ふつうの`if`文でObservableを選んでもかまいません。`iif`を選ぶのは、購読のたびに条件を評価し直したい場合や、`defer`と同じく実行を遅らせたい場合です。

## EMPTY・NEVER・throwError

値を流さない、あるいはエラーだけを流す、特殊なObservableがあります。3つまとめて見ておきましょう。

値を1つも流さずに、すぐ`complete`するのが`EMPTY`です。値も`complete`も流さず、ずっと何もしないのが`NEVER`です。そして、エラーだけを流すのが`throwError`です。

```ts
import { EMPTY, NEVER, throwError } from 'rxjs';

EMPTY.subscribe({
  next: () => console.log('呼ばれない'),
  complete: () => console.log('すぐ完了'),
});
// 出力: すぐ完了

throwError(() => new Error('失敗')).subscribe({
  error: (e) => console.log('error:', e.message),
});
// 出力: error: 失敗
```

3つの違いを表にまとめます。

| Observable | next | complete | error |
|---|---|---|---|
| `EMPTY` | なし | すぐ | なし |
| `NEVER` | なし | なし | なし |
| `throwError(...)` | なし | なし | あり |

`throwError`には、エラーオブジェクトを直接ではなく、エラーを返す関数を渡します。関数にするのは、購読のたびに新しいエラーを生成するためで、生成時に余計な副作用が起きるのを避ける狙いがあります。

## 条件分岐でObservableを返す

`EMPTY`は、「この場合は何もしない」を表すのに便利です。関数がObservableを返す約束になっているとき、何もしたくない分岐で`EMPTY`を返します。

```ts
import { EMPTY, of } from 'rxjs';

function search(keyword: string) {
  if (keyword === '') {
    return EMPTY; // 空文字なら検索しない（何も流さず完了）
  }
  return of([`${keyword}の結果`]);
}
```

ここで`null`や`undefined`を返すと、購読する側が「値があるか、ないか」の場合分けを迫られて面倒です。かわりに`EMPTY`を返せば、返り値は常にObservableになり、購読する側は、どの分岐でも同じように扱えます。エラーを表したいなら、`throwError`が同じ役割を果たします。「何もしない」も「エラー」も、Observableとして返せるわけです。

## ObservableをPromiseへ変換する

ここからは、PromiseとObservableのあいだの変換です。RxJSのコードを、`async` / `await`の世界とつなぎたいことがあります。1つの値だけが必要なら、ObservableをPromiseへ変換できます。

`firstValueFrom`は、最初に流れた値でPromiseを解決し、購読を解除します。`lastValueFrom`は、`complete`したときの、最後の値で解決します。

```ts
import { firstValueFrom, lastValueFrom, of } from 'rxjs';

const first = await firstValueFrom(of(1, 2, 3)); // 1
const last = await lastValueFrom(of(1, 2, 3)); //  3
```

HTTPリクエストのように、1回だけ結果が返るObservableを`await`したいとき、この2つが役立ちます。名前のとおり、最初の値がほしいか、最後の値がほしいかで選びます。

## completeしないObservableの問題

Promiseへの変換には、注意点があります。`lastValueFrom`は`complete`を待つので、`complete`しないObservableに使うと、Promiseは永遠に解決しません。

```ts
import { lastValueFrom, interval } from 'rxjs';

// intervalはcompleteしないので、このawaitは終わらない
const value = await lastValueFrom(interval(1000)); // 解決しない
```

`interval`のように終わらないストリームは、`lastValueFrom`と相性が悪いのです。この場合は、`take`などで途中で終わらせるか、`firstValueFrom`で最初の1つだけを取ります。初学者がawaitで固まる原因の1つが、これです。

もう1つ、値を1つも流さずに`complete`したObservable（`EMPTY`など）を変換すると、`firstValueFrom`と`lastValueFrom`はエラーになります。「値がないのに、値を取り出そうとした」からです。既定値が必要なら、第2引数で指定できます。

```ts
import { firstValueFrom, EMPTY } from 'rxjs';

const value = await firstValueFrom(EMPTY, { defaultValue: '既定値' }); // 既定値
```

## toPromiseが使われなくなった理由

以前のRxJSには、`toPromise`というメソッドがありました。RxJS 7で非推奨になり、8で削除されています。古いコードで見かけることがあるので、なぜ消えたのかを知っておきましょう。

`toPromise`には、動きがわかりにくい、という問題がありました。解決されるのは最後の値なのか最初の値なのか、値がないときはどうなるのか。名前からはまったく読み取れなかったのです（実際には、最後の値で解決し、値がなければ`undefined`で解決していました）。

その反省から生まれたのが`firstValueFrom`と`lastValueFrom`です。名前を見れば、動きがわかります。最初の値がほしいのか、最後の値がほしいのかを、書く時点で選べます。古いコードで`toPromise`を見かけたら、この2つへ置き換えてください。移行の詳しい手順は、付録の移行ガイドで扱います。

## Promiseへ変換しないほうがよい場面

最後に、逆の注意です。変換すべきでない場面もあります。ObservableをPromiseへ変換すると、いくつかの力を失うからです。

- 複数の値を扱えなくなります。Promiseは1つの値しか持てません。
- キャンセルできなくなります。Promiseには、購読解除にあたる仕組みがありません。
- Operatorでの合成ができなくなります。

ですから、途中の処理でPromiseへ変換するのは避けます。変換するのは、`async` / `await`のコードへ渡す、最後の一歩だけにします。ストリームとして扱える範囲は、できるだけObservableのまま組み立てる。これが、RxJSの力を最大限に活かす書き方です。

## Promiseへの変換はストリームの境界に限る

特殊なObservableとPromise相互変換の要点を整理します。

- `defer`は購読時にObservableを作り、値や処理の確定を遅らせます。
- `iif`は購読時に条件を評価し、2つのObservableを切り替えます。
- `EMPTY`は値なしで完了、`NEVER`は何もしない、`throwError`はエラーだけを流します。
- 何もしない分岐では`EMPTY`を返すと、返り値を常にObservableにそろえられます。
- `firstValueFrom`と`lastValueFrom`でPromiseへ変換します。`toPromise`は使いません。
- 変換はコードの境界だけにとどめ、途中はObservableのまま組み立てます。

次章からは、いよいよOperatorを本格的に扱います。まず`pipe`の仕組みと、値の流れを表すMarble Diagramの読み方を身につけます。
