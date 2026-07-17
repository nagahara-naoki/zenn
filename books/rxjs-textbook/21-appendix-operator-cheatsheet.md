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
| 条件でObservableを選ぶ | `iif` |
| 値を通知せず完了する | `EMPTY` |
| 値も完了も通知しない | `NEVER` |
| エラーを通知する | `throwError` |

作成については、「Observableを作る」と「特殊なObservableとPromise相互変換」の章で扱いました。

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

変換と選別については、「値を変換・選別・蓄積する」の章で扱いました。

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
| 一定時間ごとの最新値を使う | `auditTime` |
| 一定間隔で最新値を取得する | `sampleTime` |
| 通知を遅らせる | `delay` |
| タイムアウトさせる | `timeout` |

件数と時間の制御は、「件数と時間を制御する」の章で扱いました。

## Observableを結合する

| 目的 | Operator・Creation Function |
|---|---|
| 最新値を組み合わせる | `combineLatest` |
| 主ストリームに最新値を加える | `withLatestFrom` |
| すべての完了を待つ | `forkJoin` |
| 到着順に流す | `merge` |
| 順番に実行する | `concat` |
| 対応する値を組み合わせる | `zip` |
| 最初のObservableを採用する | `race` |

結合は、「最新値を組み合わせる」と「完了待ちと到着順」の章で扱いました。

## 非同期処理を平坦化する

| 目的 | Operator |
|---|---|
| 並行して実行する | `mergeMap` |
| 順番に実行する | `concatMap` |
| 最新の処理だけ残す | `switchMap` |
| 実行中の新しい処理を無視する | `exhaustMap` |

平坦化は、「Higher-order ObservableとNested Subscribe」と「Flattening Operator」の章で扱いました。使い分けの結論は、読み取りは`switchMap`、書き込みは`concatMap`か`exhaustMap`です。

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
| 最新値をキャッシュして共有する | `shareReplay` |

エラー・終了・共有は、「エラー処理・再試行・キャンセル」と「shareとshareReplay・Subjectによる状態管理」の章で扱いました。

## PromiseとObservableを変換する

| 目的 | 関数 |
|---|---|
| PromiseからObservableを作る | `from` |
| 最初の値をPromiseで受け取る | `firstValueFrom` |
| 最後の値をPromiseで受け取る | `lastValueFrom` |

これらは「特殊なObservableとPromise相互変換」の章で扱いました。
