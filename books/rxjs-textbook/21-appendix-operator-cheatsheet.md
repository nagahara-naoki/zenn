---
title: "付録A Operator早見表"
---

本書で扱ったOperatorとCreation Functionを、目的別にまとめます。「何がしたいか」から使うものを引けるように並べました。学習中や実務中の索引として使ってください。

それぞれの詳しい説明は、本文の該当する章にあります。動きを忘れたときは、この表で名前を確認し、章に戻って確かめると定着します。

## 値を作成する

| 目的 | Operator・Creation Function |
|---|---|
| 値からObservableを作る | `of` |
| 配列やPromiseから作る | `from` |
| イベントから作る | `fromEvent` |
| 一定間隔で値を作る | `interval` |
| 指定時間後に値を作る | `timer` |
| 購読時にObservableを作る | `defer` |
| 購読解除で中断できるHTTPを作る | `fromFetch` |
| 条件でObservableを選ぶ | `iif` |
| 値を通知せず完了する | `EMPTY` |
| 値も完了も通知しない | `NEVER` |
| エラーを通知する | `throwError` |

作成については、[「Observableを作る」](./08-creating-observables)と[「特殊なObservableとPromise相互変換」](./09-special-observables-and-promise-interop)で扱いました。`fromFetch`は[「エラー処理・再試行・キャンセル」](./19-error-retry-cancel)で扱います。

## 値を変換・選別する

| 目的 | Operator |
|---|---|
| 値を変換する | `map` |
| 条件に合う値だけを通す | `filter` |
| 副作用を実行する | `tap` |
| 連続する重複値を除外する | `distinctUntilChanged` |
| 先頭に初期値を加える | `startWith` |
| 値を蓄積する | `scan` |
| 完了時に集計する | `reduce` |

変換と選別については、[「値を変換・選別・蓄積する」](./11-transform-filter-accumulate)で扱いました。

## 件数を制御する

| 目的 | Operator |
|---|---|
| 先頭の指定件数だけ通して完了する | `take` |
| 先頭を読み飛ばす | `skip` |
| 最初の値だけ取得する | `first` |
| 完了時の最後の値を取得する | `last` |
| 条件を満たす間だけ通す | `takeWhile` |
| 別のObservableを合図に完了する | `takeUntil` |

## 時間を制御する

| 目的 | Operator |
|---|---|
| 入力が止まるまで待つ | `debounceTime` |
| 一定時間内の連続通知を抑える | `throttleTime` |
| 最初の通知から一定時間後に、その間の最新値を流す | `auditTime` |
| 固定間隔の時点で、そこまでの最新値を流す | `sampleTime` |
| `next`と`complete`を遅らせる | `delay` |
| 最初の通知や通知間隔に上限を設ける | `timeout` |

件数と時間の制御は、[「件数と時間を制御する」](./12-count-and-time-operators)で扱いました。

## Observableを結合する

| 目的 | Operator・Creation Function |
|---|---|
| 全員の初回通知後、最新値を組み合わせる | `combineLatest` |
| 主ストリームに最新値を加える | `withLatestFrom` |
| すべての完了を待ち、最後の値を集める | `forkJoin` |
| 到着順に流す | `merge` |
| 順番に実行する | `concat` |
| 対応する値を組み合わせる | `zip` |
| 最初に通知したObservableを採用する | `race` |

結合は、[「最新値を組み合わせる」](./13-combining-latest-values)と[「完了・並行・順序で合成する」](./14-forkjoin-merge-concat-zip-race)で扱いました。`combineLatest`は全員が最初の値を出すまで通知せず、`forkJoin`は値なしで完了するsourceがあると結果を出しません。`race`では`next`だけでなく、最初の`error`や`complete`も勝者を決めます。

## 非同期処理を平坦化する

| 目的 | Operator |
|---|---|
| 並行して実行する | `mergeMap` |
| 順番に実行する | `concatMap` |
| 最新の処理だけ残す | `switchMap` |
| 実行中の新しい処理を無視する | `exhaustMap` |

平坦化は、[「Higher-order ObservableとNested Subscribe」](./15-higher-order-observable)と[「Flattening Operator」](./16-flattening-operators)で扱いました。「読み取りか書き込みか」だけで決めず、すべて処理するか、順番を守るか、古いInnerを解除してよいか、処理中の新しい値を捨ててよいかで選びます。

## エラー・終了を扱う

| 目的 | Operator |
|---|---|
| エラーを処理する | `catchError` |
| 再試行する | `retry` |
| 終了時に必ず処理する | `finalize` |

## 共有する

| 目的 | Operator |
|---|---|
| 実行を共有する | `share` |
| 過去の通知を再配信しながら共有する | `shareReplay` |

エラー・終了・共有は、[「エラー処理・再試行・キャンセル」](./19-error-retry-cancel)と[「shareとshareReplay・Subjectによる状態管理」](./18-share-sharereplay-state)で扱いました。

## PromiseとObservableを変換する

| 目的 | 関数 |
|---|---|
| PromiseからObservableを作る | `from` |
| 最初の値をPromiseで受け取る | `firstValueFrom` |
| 最後の値をPromiseで受け取る | `lastValueFrom` |

これらは[「特殊なObservableとPromise相互変換」](./09-special-observables-and-promise-interop)で扱いました。

`from(fetch(...))`は、すでに始まったPromiseを包むため、購読解除で通信を中断できません。購読ごとに処理を始めるなら`defer`、HTTPの中断まで必要なら`fromFetch`を検討します。

## 段階別の総合チェック

本文を一段階読むごとに、該当する問いへ答えてください。コードを書く前に出力や購読期間を予測し、そのあと実行して確かめると理解が定着します。

1. **RxJSの基礎**: PromiseとObservableは、複数の値、実行開始、中断の3点でどう違いますか。
2. **Observableの仕組み**: 同じ`interval(1000)`を2回購読すると、タイマーはいくつ作られますか。1人が解除すると、もう1人も止まりますか。
3. **Observableの作成**: `from(fetch(url))`、`defer(() => from(fetch(url)))`、`fromFetch(url)`の違いを説明してください。
4. **Operatorの基本**: 入力文字列を前後の空白除去→空文字の除外→同じ値の連続除外→300ミリ秒の待機、の順に処理するOperator Chainを書いてください。
5. **Observableの合成**: 検索条件の最新値、複数HTTPの完了待ち、最新検索への切り替えには、それぞれ何を使いますか。
6. **Subjectと共有**: `share`、`shareReplay`、`BehaviorSubject`は、新しい購読者へ何を渡すか、外部から値を更新できるかでどう違いますか。
7. **エラー・キャンセル・テスト**: `switchMap`内の検索だけを回復させる`catchError`の位置、`retry(3)`の最大試行数、unsubscribeを表すMarbleの期待値に`|`を書かない理由を説明してください。

:::details 解答の要点
1. Promiseは1回の結果を表し、作成時に始まります。Observableは複数通知を表せますが、開始時点と中断可否はsourceの実装によります。
2. Coldな`interval`なので2つです。購読は独立しており、片方の解除はもう片方へ影響しません。
3. `from(fetch(url))`は作成時に1回始まったPromiseを観察します。`defer`版は購読ごとにfetchを始めますが、解除してもfetchは止まりません。`fromFetch`は購読時に始まり、完了前の解除をAbortControllerへ連動させます。
4. `map(trim)`→`filter(Boolean)`→`distinctUntilChanged()`→`debounceTime(300)`が一例です。空白を除去してから空文字と重複を判定します。
5. `combineLatest`、`forkJoin`、`switchMap`です。それぞれ初回通知、完了、Innerの解除という端条件も確認します。
6. `share`は共有後の通知だけ、`shareReplay`は指定件数の過去通知も配ります。どちらも通常は外部から`next`しません。`BehaviorSubject`は現在値を持ち、外部から更新できるため、公開範囲を制限します。
7. `catchError`は`switchMap`が返すInnerの中です。`retry(3)`は初回を含め最大4試行です。unsubscribeはcomplete通知ではないため、期待値に`|`を書きません。
:::
