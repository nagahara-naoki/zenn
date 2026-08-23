---
title: "付録C RxJS用語集"
---

本書で使う中心用語を、英字順にまとめます。Operatorを目的から探したい場合は、付録Aの「Operator早見表」を使ってください。

## BehaviorSubject

初期値と現在値を持つSubject。購読すると、その時点の最新値をすぐに受け取り、その後の通知も受け取る。小規模な状態管理では、Subject自体を外へ公開せず、`asObservable()`で読み取り側だけを公開する。

## catchError

上流の`error`通知を受け、代わりとなるObservableへ切り替えるPipeable Operator。Flattening Operatorの外側に置くと、その出力全体を差し替える。Innerの中に置くと、1回分の失敗だけを処理できる。

## Cold Observable / Hot Observable

**Cold Observable**は、購読をきっかけにProducerを作る、または処理を開始するObservable。`of`や`interval`、`defer(() => from(fetch(...)))`などが該当し、購読ごとに独立した実行になりやすい。

**Hot Observable**は、購読とは別に動くProducerを観測するObservable。DOMイベントや、接続済みWebSocketの通知などが該当する。`fromEvent`は購読ごとにイベントリスナーを登録するが、イベントを生むDOM自体は購読の外で動いている。

Cold / Hotは「Producerがいつ、どこで動くか」の区別で、Unicast / Multicastとは別の軸である。

## complete / error / next

ObservableがObserverへ渡す3種類の通知。`next`は値を表し、0回以上流せる。`complete`は正常終了、`error`は異常終了を表す。`complete`と`error`はどちらか一度だけで、そのあとに通知は流れない。

## Creation Function

Observableを新しく作る関数。`of`、`from`、`fromEvent`、`interval`、`defer`など。source Observableを受け取るPipeable Operatorとは役割が異なる。

## finalize

`complete`、`error`、明示的な`unsubscribe`のどの経路でも、購読の終了時にコールバックを実行するOperator。ローディング表示の解除など、終了理由によらない後始末に使う。

## firstValueFrom / lastValueFrom

ObservableをPromiseとして待つ関数。`firstValueFrom`は最初の値を受け取ると購読を解除し、`lastValueFrom`は完了まで待って最後の値を返す。値なしで完了すると拒否され、終わらないObservableを`lastValueFrom`へ渡すとPromiseも完了しない。

## Flattening / Higher-order Observable

**Higher-order Observable**は、値として別のObservableを流すObservable。外側をOuter Observable、値として流れてくる内側をInner Observableと呼ぶ。

**Flattening（平坦化）**は、Innerを購読し、その通知を1本のObservableへまとめること。`mergeMap`、`concatMap`、`switchMap`、`exhaustMap`は、新しいInnerが来たときの扱いが異なる。

## Marble Diagram

値、時間、完了、エラー、購読期間を横方向に表す図。本文の説明用記法とTestSchedulerのテスト用記法では、エラーなど一部の記号が異なる。

## Multicast / Unicast

**Unicast**は、通知を受け取る経路や実行が購読者ごとに独立している配信。**Multicast**は、1つのsource購読からの通知を複数の購読者へ配る配信。`Subject`や`share`でMulticastを作れる。

ColdとUnicast、HotとMulticastはよく組み合わさるが、同義ではない。Cold / HotはProducer、Unicast / Multicastは通知の共有に注目した分類である。

## Observable

`next`、`error`、`complete`という通知をObserverへ届ける仕組み。`subscribe`によってObserverとの接続を作る。Cold Observableでは購読をきっかけに処理が始まるが、Hot ObservableのProducerは購読前から動いていることがある。

## Observer / Subscriber

**Observer**は通知を受け取る側で、`next`、`error`、`complete`のコールバックを持つ。実際の`subscribe`では、必要なコールバックだけを持つPartial Observerも渡せる。

**Subscriber**は、Observerを包んで通知規約と購読状態を管理するRxJS内部の存在。終了後の通知を通さず、Subscriptionとしての解除も扱う。

## Operator / Pipeable Operator

本書で単に**Operator**と呼ぶものは、Observableを受け取り、新しいObservableを返すPipeable Operatorを指す。`map`、`filter`、`switchMap`などが該当する。Observableを新しく作る`of`や`from`はCreation Functionである。

## pipe

Pipeable Operatorを左から右へ順に適用し、変換後のObservableを返すメソッド。元のObservableそのものを書き換えるわけではない。

## Producer / Consumer

**Producer**は値や通知を生む側。タイマー、DOM、HTTPクライアントなどが該当する。**Consumer**は通知を受け取る側で、RxJSではObserverが担う。Push型ではProducer側のタイミングで通知が届き、Pull型ではConsumer側が次の値を要求する。

## refCount

共有されたObservableの購読者数を数え、最初の購読でsourceへ接続し、0人になったときに接続を解除する考え方。`shareReplay`では、`refCount: true`を明示して使う。

## retry

エラー時にsourceを再購読するOperator。`retry(3)`は初回を含む3試行ではなく、初回に加えて最大3回再購読する。実務では、上限、待ち時間、jitter、処理の冪等性を合わせて設計する。

## Scheduler / TestScheduler

**Scheduler**は、RxJSの仕事をいつ実行するかを調整する仕組み。**TestScheduler**はSchedulerで管理される時間を仮想化し、時間依存のOperatorをMarble Testで検証する。任意のPromiseや実際のHTTP通信まで自動で仮想化するものではない。

## share / shareReplay

**share**は、複数の購読者で1つのsource購読を共有するOperator。通常、購読者が0になるとsourceとの接続を解除する。

**shareReplay**は共有に加えて、指定した件数の過去通知を新しい購読者へ再配信する。`refCount`、キャッシュの更新条件、sourceが完了するかを意識して使う。

## Subject

Observableであると同時にObserverでもあり、外部から`next`、`error`、`complete`を渡せる。1つの通知を複数の購読者へ配信する。基本の`Subject`に加えて、`BehaviorSubject`、`ReplaySubject`、`AsyncSubject`がある。

## subscribe / Subscription / unsubscribe

**subscribe**はObservableとObserverを接続し、Subscriptionを返す。Cold Observableでは、この接続をきっかけにProducerが作られる。Hot Observableでは、すでに動いているProducerの通知を受け始める。

**Subscription**は購読を表すオブジェクト。**unsubscribe**を呼ぶと通知の受け取りをやめ、登録されたTeardown Logicを実行する。unsubscribeは`complete`通知ではなく、Producerの処理自体が止まるかどうかはTeardown Logicに依存する。

## Teardown Logic

購読の終了時に実行する後片付け。タイマーの停止、イベントリスナーの解除、AbortControllerによる通信中断などを書く。`from(fetch(...))`はfetchを中断しないが、`fromFetch`は購読解除とAbortControllerを連動させる。
