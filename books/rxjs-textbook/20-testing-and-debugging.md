---
title: "テストとデバッグ"
---

本書の締めくくりとして、テストとデバッグを扱います。

RxJSのコードは、時間が絡むため、テストが難しく感じられます。「1秒後に値が流れる」処理を確かめるのに、実際に1秒待つテストを書いていたら、テストはどんどん遅くなり、しかも不安定になります。この問題を解決するのが、仮想時間を使うTestSchedulerです。仮想時間なら、1秒も1時間も、一瞬で進められます。

前半でTestSchedulerによるテストを、後半で`tap`を使ったデバッグと、よくある問題の見つけ方を扱います。ここまで学んだOperatorが、実際に意図どおり動くかを確かめる方法を、身につけましょう。

## テストで難しいのは時間

まず、なぜRxJSのテストが難しいのかを、はっきりさせておきます。理由は、ほとんどの場合、時間です。

`debounceTime(300)`のOperatorをテストするとき、本当に300ミリ秒待っていたら、テストが遅くなります。`interval`を使う処理なら、何秒も待つことになります。テストは何百個も走らせるものなので、1つあたりの待ち時間は、積もると大きな問題になります。時間に依存する処理を、実時間でテストするのは、効率が悪く、結果も不安定になりがちです。

そこでRxJSは、時間を仮想的に進める仕組みを用意しています。それがTestSchedulerです。

## TestSchedulerと仮想時間

TestSchedulerは、時間を仮想的に扱うSchedulerです。「ColdとHot・同期と非同期」の章で名前だけ触れたSchedulerが、ここで役割を持ちます。TestSchedulerを使うと、実際には待たず、時間を一瞬で進めてテストできます。「300ミリ秒待つ」処理も、待ち時間ゼロで検証できるのです。

```ts
import { TestScheduler } from 'rxjs/testing';

const testScheduler = new TestScheduler((actual, expected) => {
  // 実行結果と期待値を比較する（テストフレームワークのassertを使う）
  expect(actual).toEqual(expected);
});
```

TestSchedulerは、`rxjs`本体ではなく`rxjs/testing`からimportします。生成するときに、「実行結果と期待値を比較する関数」を渡します。

## テスト向けのMarble記法

TestSchedulerでは、値の流れを、Marble Diagramの文字列で書きます。「Operatorとpipe・Marble Diagramの読み方」の章で見た読みやすい記法に、テスト用の記号がいくつか加わります。

| 記号 | 意味 |
|---|---|
| ` `（空白） | 何もしない（桁そろえのためだけ。時間は進まない） |
| `-` | 1フレーム（仮想的な時間の1単位）の経過 |
| `a` `b` | その位置で値を流す |
| `\|` | complete |
| `#` | error |
| `(ab)` | 同じフレームで複数の通知 |
| `^` | 購読の開始（Hot Observable） |
| `!` | 購読の解除 |

読みやすさの記法との違いは、2つです。エラーが`X`から`#`に変わること。そして、購読の開始`^`と解除`!`が加わることです。テストでは、購読のタイミングまで検証したいので、この2つの記号が増えています。

## Operatorをテストする

実際に、`map`をテストしてみましょう。`testScheduler.run`の中で、入力を`cold`で作り、結果を`expectObservable`で検証します。

```ts
import { map } from 'rxjs';

testScheduler.run(({ cold, expectObservable }) => {
  const source$ = cold('--a--b--|', { a: 1, b: 2 });
  const result$ = source$.pipe(map((n) => n * 10));

  expectObservable(result$).toBe('--x--y--|', { x: 10, y: 20 });
});
```

読み解いてみましょう。`cold('--a--b--|', { a: 1, b: 2 })`が入力です。`a`が値`1`、`b`が値`2`を表します。それに`map`で10倍をかけた結果が、`--x--y--|`（`x`が`10`、`y`が`20`）になることを検証しています。入力と出力を、Marble Diagramで並べて書くだけで、テストが書けるのです。文章で「1が10に変換される」と書くより、ずっと直感的です。

## 非同期処理を同期的に検証する

時間の絡むOperatorも、仮想時間なら一瞬でテストできます。`debounceTime`をテストしてみます。

```ts
import { debounceTime } from 'rxjs';

testScheduler.run(({ cold, expectObservable }) => {
  const source$ = cold('a-b-c-------|');
  const result$ = source$.pipe(debounceTime(3));

  // 最後のcのあと、3フレーム静かになってからcが流れる
  expectObservable(result$).toBe('--------c---|');
});
```

`debounceTime(3)`は、値が止まってから3フレーム待って流します。入力を見ると、`a`、`b`、`c`と続いたあと、静かになっています。だから、出力では、`c`のあとで静かになってから、`c`が1つだけ流れます。実際には一切待たずに、この「時間の挙動」を検証できているのが、TestSchedulerのすごいところです。

## Cold・HotとSubscriptionの記法

`cold`はCold Observable、`hot`はHot Observableを作ります。Hotでは、`^`で購読の開始位置を示します。

購読が、いつ始まっていつ終わるかも、`expectSubscriptions`で検証できます。購読のMarbleでは、`^`が購読開始、`!`が購読解除です。

```ts
testScheduler.run(({ cold, expectObservable, expectSubscriptions }) => {
  const source$ = cold('--a--b--c--|');
  const subs = '^------!'; // ここで購読し、ここで解除される想定

  expectObservable(source$, subs).toBe('--a--b--|');
  expectSubscriptions(source$.subscriptions).toBe(subs);
});
```

これが役立つのは、`switchMap`が古いInner Observableを解除する、といった「購読の解除がからむ挙動」を検証したいときです。目に見えない購読の動きを、図として確かめられます。

## Marble Testを読み解く

Marble Testは、慣れるまでは記号の羅列に見えて、とっつきにくいかもしれません。読み解くコツは、入力・出力・購読を上下に並べて、同じ桁を縦に見ることです。

```text
入力:   --a--b--|
出力:   --x--y--|
```

同じ桁を見ると、`a`の位置で`x`が、`b`の位置で`y`が流れているとわかります。Operatorが、値の位置を変えるのか、時間をずらすのか、値を間引くのか。それが、桁の対応から読み取れます。他人が書いたテストを読むときも、まず桁をそろえて眺めると、意図がつかめます。

## tapによるデバッグ

ここからはデバッグです。ストリームの途中で何が起きているかを見るには、「値を変換・選別・蓄積する」の章で扱った`tap`が役立ちます。

RxJS 7の`tap`は、`next`（値）に加えて、購読の開始や解除も観察できます。

```ts
import { tap } from 'rxjs';

source$.pipe(
  tap({
    subscribe: () => console.log('購読開始'),
    next: (value) => console.log('値:', value),
    error: (error) => console.log('エラー:', error),
    complete: () => console.log('完了'),
    unsubscribe: () => console.log('購読解除'),
    finalize: () => console.log('終了'),
  }),
);
```

これで、いつ購読が始まり、どの値が流れ、いつ終わったかが、すべてログに出ます。「値が流れない」「いつまでも終わらない」といった不具合の原因を探るのに、とても役立ちます。目に見えないストリームの動きを、ログで可視化するわけです。

## 値の流れと購読を追跡する

`tap`は、Operator Chainの好きな位置に挟めます。変換の前後に置けば、その段階での値を確認できます。

```ts
source$.pipe(
  tap((v) => console.log('変換前:', v)),
  map((v) => v * 10),
  tap((v) => console.log('変換後:', v)),
);
```

こうすると、「どのOperatorで値がおかしくなったのか」を、段階ごとに切り分けられます。バグの発生源を、しらみつぶしに探せるわけです。なお、デバッグが終わったら、`tap`は忘れずに外しておきましょう。

## よくある問題を見つける

RxJSでつまずきやすい問題は、だいたい決まっています。デバッグの勘どころを、まとめて挙げておきます。

**多重購読と多重HTTPリクエスト**。同じObservableを、複数回購読していないかを疑います。購読開始のログが想定より多く出るなら、多重購読です。HTTPなら、そのぶんリクエストが飛んでいます。`shareReplay`での共有を検討します。

**メモリリーク**。購読解除のログが出ないまま画面が変わっているなら、購読が残っています。`unsubscribe`や`takeUntil`で、きちんと解除できているかを確認します。

**Observableがcompleteしない**。`complete`のログが出ないなら、そのObservableは終わっていません。`forkJoin`や`lastValueFrom`が動かない原因は、たいていこれです。

**shareReplayによるキャッシュ問題**。古いデータが表示され続けるなら、`shareReplay`のキャッシュが更新されていないのかもしれません。あわせて、`refCount`の設定漏れによる、購読の残留も疑います。

## Operator Chainを分割する

デバッグのしやすさと、読みやすさのために、長いOperator Chainは分割します。1つの`pipe`にすべてを詰め込むと、どこで問題が起きたのかを、追いにくくなるからです。

```ts
// 意味のまとまりで名前を付けて分ける
const keyword$ = input$.pipe(
  map((e) => (e.target as HTMLInputElement).value),
  debounceTime(300),
  distinctUntilChanged(),
);

const results$ = keyword$.pipe(
  switchMap((keyword) => searchApi(keyword)),
);
```

「入力を整えるところ（`keyword$`）」と「検索するところ（`results$`）」を分けて名前を付けると、それぞれを個別に確認でき、意図も読み取りやすくなります。「Operatorとpipe・Marble Diagramの読み方」の章で触れた「読みやすいOperatorの並べ方」を、テストとデバッグの観点からも実践する、というわけです。

## 仮想時間と観測点がRxJSのテストを安定させる

テストとデバッグで使う手段を整理します。

- RxJSのテストが難しいのは時間が絡むためで、TestSchedulerの仮想時間で解決します。
- TestSchedulerは`rxjs/testing`から使い、Marble記法で入力と出力を並べて検証します。
- テスト向けの記法では、エラーは`#`、購読開始は`^`、購読解除は`!`で表します。
- `tap`は`next`に加えて購読の開始・解除も観察でき、デバッグに役立ちます。
- 多重購読、メモリリーク、completeしない問題、キャッシュ問題は、ログで見つけられます。
- 長いOperator Chainは、意味のまとまりで分割すると、テストもデバッグもしやすくなります。

これで本編は終わりです。RxJSの仕組みから、Operatorの選択、合成、共有、エラー処理、テストまでを、一通り見てきました。巻末の付録には、Operatorの早見表、古い書き方からの移行ガイド、用語集を用意しています。学習と実務の両方で、索引として役立ててください。
