# RxJSの教科書

## 非同期処理をストリームで設計する

---

## 本書について

RxJSは、非同期処理やイベントを「時間とともに流れる値」として扱うためのライブラリです。

AngularのHttpClientやフォームをはじめ、フロントエンド開発の多くの場面でRxJSが使われています。しかし、RxJSには数多くのOperatorが存在するため、次のような悩みを持つ人も少なくありません。

- Observableの仕組みがよく分からない
- `subscribe`の中に`subscribe`を書いてしまう
- `switchMap`、`mergeMap`、`concatMap`、`exhaustMap`の違いが分からない
- Subjectをどこで使うべきか判断できない
- 購読解除が必要な場面を理解できていない

本書では、Operatorを暗記するのではなく、RxJSの基本構造から学び、非同期処理をストリームとして設計できるようになることを目指します。

```text
非同期処理の問題
    ↓
Observableの仕組み
    ↓
Operatorによる変換
    ↓
複数ストリームの合成
    ↓
Subjectと共有
    ↓
エラー・キャンセル・テスト
```

本書はRxJSそのものを扱います。Angular・NgRxでのRxJS実践は、シリーズ後続の書籍で扱います。

本書のサンプルコードは、原則としてTypeScriptで記述します。RxJSはバージョン7.8系（Angularが依存する現行の安定バージョン）を基準とします。

---

## 対象読者

本書は、次のような読者を対象とします。

- JavaScriptまたはTypeScriptの基本文法を理解している人
- Promiseや`async` / `await`を使ったことがある人
- RxJSを基礎から学び直したい人
- RxJSのOperatorを正しく選べるようになりたい人
- Angular学習の前提として、RxJSを固めておきたい人

---

## 本書の到達目標

本書を読み終えることで、次のことができるようになります。

- Observable、Observer、Subscriptionの関係を説明できる
- Cold ObservableとHot Observableを区別できる
- Marble Diagramを読める
- Operatorを目的に応じて選択できる
- Higher-order Observableを理解できる
- Nested Subscribeを適切なOperatorへ置き換えられる
- Subjectの種類と用途を説明できる
- エラー処理、再試行、キャンセルを設計できる
- TestSchedulerでOperatorの動作を検証できる

---

# 第1部 RxJSの基礎

## 第1章 非同期処理とリアクティブプログラミング

- 同期処理と非同期処理
- ユーザー操作・HTTP通信・タイマー・WebSocket
- 複数の非同期処理が組み合わさる問題
- コールバックによる処理
- Promiseによる処理
- Promiseだけでは表現しにくい処理
- キャンセル可能な非同期処理
- 時間とともに複数の値を受け取る処理
- リアクティブプログラミングの考え方
- 命令型プログラミングと宣言型プログラミング
- 値の変化に反応する処理
- データの流れを組み立てる

## 第2章 ストリームとRxJSの全体像

- 時間とともに流れる値
- 単一の値と複数の値
- ストリームの開始・値の通知・エラーの通知・完了の通知
- DOMイベントをストリームとして考える
- HTTP通信をストリームとして考える
- タイマーをストリームとして考える
- RxJSとは
- Observable・Observer・Subscription
- Operator・Creation Function
- Subject・Scheduler
- RxJSの処理の流れ

## 第3章 はじめてのRxJS

- RxJSのインストール
- Observableを作成する
- `subscribe`で値を受け取る
- `pipe`でOperatorをつなぐ
- `map`で値を変換する
- `filter`で値を絞り込む
- `unsubscribe`で購読を解除する
- 最初のRxJSプログラムを作る

---

# 第2部 Observableの仕組み

## 第4章 Observable・Observer・subscribeの仕組み

- Observableの役割
- 値を生成するProducerと値を受け取るConsumer
- Pull型とPush型
- Observableは値そのものではなく処理の設計図
- 関数との違い・Iterableとの違い・Promiseとの違い
- Observerの役割
- `next`・`error`・`complete`
- Partial Observer
- `error`と`complete`の後に値が流れない理由
- `subscribe`すると何が起きるのか
- SubscriberとSubscriptionの生成
- 値が通知される流れ
- 複数回`subscribe`した場合
- Observableの遅延実行

## 第5章 Subscription・購読解除・Observableの自作

- Subscriptionとは
- `unsubscribe`
- Teardown Logic
- タイマーの停止・イベントリスナーの解除
- 複数のSubscriptionをまとめる
- 不要な購読が残る問題とメモリリーク
- `new Observable`でObservableを自作する
- `next`・`error`・`complete`で通知する
- Teardown処理を書く
- タイマーをObservableにする
- イベントをObservableにする
- 自作Observableを使う際の注意点

## 第6章 ColdとHot・同期と非同期

- Cold Observableとは
- Hot Observableとは
- 購読者ごとに実行される処理
- 複数の購読者で共有される処理
- HTTPリクエスト・DOMイベント・タイマー・WebSocket
- UnicastとMulticast（本章では現象の観察に絞り、共有の仕組みはSubjectの章で扱う）
- ColdとHotを二択だけで考えない
- Observableは常に非同期とは限らない
- `of`・`from`・`interval`の実行タイミング
- `subscribe`前後の処理順序
- 同期通知による注意点
- JavaScriptのイベントループとの関係

---

# 第3部 Observableの作成

## 第7章 Observableを作る

- `of`で値からObservableを作る
- `from`で配列・Iterable・Promise・文字列から作る
- `of`と`from`の違い
- `fromEvent`でDOMイベントからObservableを作る
- クリック・入力・スクロール・キーボード
- 購読解除によるイベントリスナーの削除
- `interval`と`timer`
- カウントダウン・定期ポーリング
- 購読解除で停止する
- タイマーの多重起動を防ぐ

## 第8章 特殊なObservableとPromise相互変換

- `defer`によるObservableの遅延生成
- 作成時と購読時の違い
- `iif`で条件によってObservableを切り替える
- `EMPTY`・`NEVER`・`throwError`
- 条件分岐でObservableを返す
- PromiseからObservableを作る
- `firstValueFrom`・`lastValueFrom`でObservableをPromiseへ変換する
- completeしないObservableの問題
- `toPromise`が使われなくなった理由
- Promiseへ変換しないほうがよい場面

---

# 第4部 Operatorの基本

## 第9章 Operatorとpipe・Marble Diagramの読み方

- Operatorとは
- Pipeable Operator
- `pipe`でOperatorをつなげる
- 入力Observableと出力Observable
- 元のObservableは変更されない
- Operator Chain
- 読みやすいOperatorの並べ方
- Marble Diagramとは
- 時間軸・値の通知・complete・errorの記法
- Marble DiagramでOperatorの動きを読む

## 第10章 値を変換・選別・蓄積する

- `map`で値を変換する
- 配列の`map`との共通点
- APIレスポンスを整形する
- `map`の中で副作用を起こさない
- `map`の中でPromiseを返した場合
- `tap`で副作用を扱う
- ログ出力とデバッグ
- `tap`で値を変更しない
- `filter`で値を絞り込む
- Type Guardとして使う
- `distinctUntilChanged`で連続した同じ値を除外する
- 参照比較と比較関数
- `scan`と`reduce`で値を蓄積する
- 途中結果の通知と完了時の通知
- イミュータブルな状態更新

## 第11章 件数と時間を制御する

- `take`・`skip`・`first`・`last`
- `take(1)`と`first()`の違い
- `takeWhile`
- 自動的にcompleteさせる
- 条件を満たさない場合の違い
- `debounceTime`で入力が止まるまで待つ
- `throttleTime`で連続通知を抑制する
- `auditTime`・`sampleTime`
- `delay`で通知を遅らせる
- インクリメンタル検索・スクロールイベント・ボタンの連打防止
- 時間制御Operatorの使い分け

---

# 第5部 Observableの合成

## 第12章 最新値を組み合わせる

- 複数のObservableを扱う
- 結合Operatorを選ぶ基準
- Creation FunctionとPipeable Operatorの違い（`combineLatestWith`など）
- `combineLatest`で最新値を組み合わせる
- 初回通知の条件
- 値が一度も通知されない場合
- `startWith`で初期値を与える
- `withLatestFrom`
- 主となるObservableと補助的なObservable
- フォーム値と状態を組み合わせる
- `combineLatest`と`withLatestFrom`の使い分け

## 第13章 完了待ちと到着順

- `forkJoin`で複数Observableの完了を待つ
- `Promise.all`との比較
- completeしないObservableの問題
- 一つでもエラーになった場合
- `merge`で到着順に流す
- `concat`で順番に実行する
- `zip`で対応する値を組み合わせる
- `race`で最初のObservableを採用する
- 各結合方法の違いと選び方

## 第14章 Higher-order ObservableとNested Subscribe

- Observableの中にObservableがある状態
- Higher-order Observable
- Inner ObservableとOuter Observable
- `map`でObservableを返すとどうなるか
- Nested Subscribeの問題
- Observableを平坦化する
- Flattening Operator
- 非同期処理の競合
- 処理順序とキャンセル

## 第15章 Flattening Operator

- `mergeMap`で並行して実行する
- 完了順に結果を受け取る・実行順序が保証されない
- 同時実行数を制限する
- `concatMap`で順番に実行する
- キューとして扱う・実行待ちが増える問題
- `switchMap`で最新の処理だけを残す
- 古いInner Observableの購読解除
- インクリメンタル検索
- `exhaustMap`で処理中の新しい値を無視する
- ログイン処理・二重送信防止
- 4つのFlattening Operatorの使い分け

| Operator | 新しい値が通知されたとき | 主な用途 |
| --- | --- | --- |
| `mergeMap` | 並行して実行する | 独立した複数処理 |
| `concatMap` | 待機させる | 順番が重要な処理 |
| `switchMap` | 前の処理を解除する | 最新の処理だけ必要な場合 |
| `exhaustMap` | 新しい処理を無視する | 二重実行を防ぎたい場合 |

---

# 第6部 Subjectと共有

## 第16章 SubjectとMulticast

- Subjectの役割
- ObservableでもありObserverでもある
- 外部から値を流す
- 複数の購読者へ値を配信する
- `Subject`・`BehaviorSubject`・`ReplaySubject`・`AsyncSubject`
- 各Subjectの使い分けと判断基準
- UnicastとMulticastの仕組み
- Cold Observableを共有する
- HTTP通信の多重実行
- 共有による副作用
- Subjectを使うべき場面・使わなくてよい場面

## 第17章 shareとshareReplay・Subjectによる状態管理

- `share`でObservableの実行を共有する
- 購読者数と`refCount`
- 最後の購読者が解除された場合
- `shareReplay`で最新値を新しい購読者へ再配信する
- レスポンスのキャッシュ
- 意図しない永続購読
- キャッシュの無効化
- `BehaviorSubject`で状態を保持する
- Subjectを外部へ公開しない
- 読み取り専用Observableと状態更新メソッド
- `scan`による状態更新
- 小規模な状態管理
- Subjectを乱用しない
- 状態管理ライブラリを使うべき境界

---

# 第7部 エラー・キャンセル・テスト

## 第18章 エラー処理・再試行・キャンセル

- Observableの`error`
- エラー後に値が通知されない理由
- `catchError`で代替値を返す・別のObservableへ切り替える
- エラーを再通知する・エラーを握りつぶさない
- `catchError`を置く位置
- Inner ObservableのエラーとOuter Observableを停止させない書き方
- `retry`で再試行する
- 再試行回数・再試行間隔・条件付き再試行
- Exponential Backoffの考え方
- `timeout`で処理を打ち切る
- 通信エラーと業務エラー・無限リトライを避ける
- `takeUntil`・`take`・`finalize`
- `switchMap`によるキャンセル
- AbortSignalとの関係
- 終了処理を確実に実行する

## 第19章 テストとデバッグ

- Cold ObservableとHot ObservableのMarble記法
- Subscriptionの記法
- `TestScheduler`と仮想時間
- Schedulerとは
- Operatorのテスト
- 非同期処理を同期的に検証する
- Marble Testを読み解く
- `tap`によるデバッグ
- 値の流れ・購読開始・購読解除を追跡する
- 多重購読と多重HTTPリクエストの発見
- メモリリーク
- Observableがcompleteしない問題
- `shareReplay`によるキャッシュ問題
- Operator Chainを分割する

---

# 付録

## 付録A Operator早見表

### 値を作成する

| 目的 | Operator・Creation Function |
| --- | --- |
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

### 値を変換・選別する

| 目的 | Operator |
| --- | --- |
| 値を変換する | `map` |
| 条件に合う値だけを通す | `filter` |
| 副作用を実行する | `tap` |
| 連続する重複値を除外する | `distinctUntilChanged` |
| 先頭に初期値を加える | `startWith` |
| 値を蓄積する | `scan` |
| 完了時に集計する | `reduce` |

### 時間を制御する

| 目的 | Operator |
| --- | --- |
| 入力が止まるまで待つ | `debounceTime` |
| 一定時間内の連続通知を抑える | `throttleTime` |
| 一定時間ごとの最後の値を使う | `auditTime` |
| 一定間隔で最新値を取得する | `sampleTime` |
| 通知を遅らせる | `delay` |
| タイムアウトさせる | `timeout` |

### Observableを結合する

| 目的 | Operator・Creation Function |
| --- | --- |
| 最新値を組み合わせる | `combineLatest` |
| 主ストリームに最新値を加える | `withLatestFrom` |
| すべての完了を待つ | `forkJoin` |
| 到着順に流す | `merge` |
| 順番に実行する | `concat` |
| 対応する値を組み合わせる | `zip` |
| 最初のObservableを採用する | `race` |

### 非同期処理を平坦化する

| 目的 | Operator |
| --- | --- |
| 並行して実行する | `mergeMap` |
| 順番に実行する | `concatMap` |
| 最新の処理だけ残す | `switchMap` |
| 実行中の新しい処理を無視する | `exhaustMap` |

### 終了・エラーを扱う

| 目的 | Operator |
| --- | --- |
| 指定件数で終了する | `take` |
| 先頭を読み飛ばす | `skip` |
| 最初の値だけ取得する | `first` |
| 完了時の最後の値を取得する | `last` |
| 条件を満たす間だけ通す | `takeWhile` |
| 別のObservableを合図に終了する | `takeUntil` |
| エラーを処理する | `catchError` |
| 再試行する | `retry` |
| 終了時に処理する | `finalize` |

---

## 付録B 非推奨・古い書き方からの移行

- `toPromise`から`firstValueFrom`または`lastValueFrom`へ移行する
- `retryWhen`から`retry`の設定オブジェクト（`retry({ delay })`）へ移行する
- `multicast`・`publish`・`refCount`から`share`・`connectable`へ移行する
- `mapTo`・`switchMapTo`などの`〜To`系Operatorから関数を渡す形式へ移行する
- 古い`subscribe`引数形式をObserverオブジェクトへ移行する
- `rxjs/operators`からのimportを`rxjs`からのimportへ移行する
- Nested SubscribeをFlattening Operatorへ置き換える
- 手動のSubscription管理を見直す
- `shareReplay`による永続購読を見直す
- `BehaviorSubject`の外部公開をやめる

---

## 付録C RxJS用語集

- Observable
- Observer
- Subscriber
- Subscription
- Notification
- Producer
- Consumer
- Push型・Pull型
- Operator
- Creation Function
- Pipeable Operator
- Subject
- Cold Observable
- Hot Observable
- Unicast
- Multicast
- Higher-order Observable
- Inner Observable
- Outer Observable
- Flattening
- Teardown Logic
- Scheduler
- Marble Diagram

---

# 本書の学習の流れ

```text
第1部
RxJSが必要になる理由と全体像を理解する
    ↓
第2部
Observableの内部構造を理解する
    ↓
第3部
Observableの作成方法を学ぶ
    ↓
第4部
値の変換・選別・時間制御を学ぶ
    ↓
第5部
複数の非同期処理を組み合わせる
    ↓
第6部
Subjectと共有を学ぶ
    ↓
第7部
エラー・キャンセル・テストを学ぶ
```

---

# 本書で重視すること

## Operatorを暗記しない

RxJSでは、Operatorの名前を覚えるだけでは実務で適切に使えません。

本書では、次の順番でOperatorを選べるようにします。

1. 何を起点に値が流れるのか
2. どの値を残すのか
3. 値をどのように変換するのか
4. 複数の処理を並行・直列・キャンセルのどれで扱うのか
5. エラーや終了をどこで扱うのか

## subscribeの位置を意識する

RxJSでは、Observableを作るだけでは処理は実行されません。

どこで購読が開始され、どこで購読が終了するのかを常に意識します。

## Subjectを安易に使わない

Subjectは便利ですが、外部から自由に値を流せるため、状態の流れを追いにくくする原因にもなります。

Observableだけで表現できないかを検討したうえで、必要な場合にSubjectを利用します。

## フレームワークに依存しない理解

特定のフレームワークのAPIとしてではなく、RxJSそのものの仕組みを理解します。

この土台があれば、Angular、NgRx、イベント処理など、どの応用先にも同じ考え方で臨めます。

---

# シリーズ内での位置付け

```text
RxJSの教科書
        ↓
Angularの教科書
        ↓
NgRxの教科書
```

『RxJSの教科書』は、フロントエンド開発に向かうシリーズの土台に位置付けます。RxJSはAngular開発の多くの場面で登場し、NgRxではさらに深く使われます。

AngularでのRxJS実践（HttpClient、購読の管理、Signalsとの連携）は『Angularの教科書』で、NgRx EffectsでのOperator選択は『NgRxの教科書』で扱います。

本書の目標は、RxJSのOperatorを覚えることではありません。

**非同期処理をストリームとして捉え、適切なOperatorを選び、保守しやすい処理を設計できるようになること**を目指します。
