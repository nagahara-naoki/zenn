---
title: "付録C RxJS用語集"
---

本書に登場したRxJSの用語を、短い説明とともにまとめます。読み進める中で意味があいまいになったとき、ここで確認してください。それぞれの用語は、本文の中で詳しく扱っています。

## 基本の登場人物

**Observable**
値をどう流すかを書いた設計図。購読するまで動かない。本書全体の主役。

**Observer**
流れてくる通知を受け取る側。`next`・`error`・`complete`の3つのメソッドを持つ。一部だけを持つものはPartial Observerと呼ぶ。

**Subscriber**
Observerを包んだ内部の存在。終わったストリームから値を流さないなど、通知の決まりを守る役目を持つ。

**Subscription**
購読を表すオブジェクト。`unsubscribe`で購読を解除できる。

**Producer**
Observableの内側で値を生み出すもの。タイマー、DOMイベント、HTTPリクエストなど。

**Consumer**
値を受け取る側。RxJSではObserverがこの役割を担う。

## 通知

**Notification**
ストリームが運ぶ通知。`next`（値）・`error`（異常終了）・`complete`（正常終了）の3種類がある。

**next / error / complete**
`next`は値の通知で何回でも起こる。`error`と`complete`は終了の通知で、一度きり。どちらかが起きると、そのあとに値は流れない。

## 方式

**Push型 / Pull型**
Push型は、生み出す側が値を送りつける方式。`Promise`やObservableがこれにあたる。Pull型は、受け取る側が要求したときに値が返る方式。関数やIteratorがこれにあたる。

## Operatorと生成

**Operator**
Observableを受け取り、新しいObservableを返す関数。値を変換・選別・合成する。

**Creation Function**
Observableを新しく作る関数。`of`、`from`、`interval`など。

**Pipeable Operator**
`pipe`の中で使うOperator。`map`や`filter`など。既存のObservableを変換する。

## Subject

**Subject**
ObservableとObserverの両方の顔を持つ。外から値を流せ、複数の購読者へ配れる。`BehaviorSubject`・`ReplaySubject`・`AsyncSubject`などの種類がある。

## ColdとHot

**Cold Observable**
購読するたびに新しい実行が始まるObservable。購読者どうしは独立する。HTTPリクエストやタイマーが該当する。

**Hot Observable**
1つの実行を購読者どうしで共有するObservable。遅れて購読すると途中から受け取る。DOMイベントが該当する。

**Unicast / Multicast**
Unicastは購読者ごとに専用の実行を届けること。Coldがこれにあたる。Multicastは1つの実行を複数へ配ること。Hotがこれにあたる。

## 平坦化

**Higher-order Observable**
値としてObservableを流すObservable。`map`の中でObservableを返すとできる。

**Inner Observable / Outer Observable**
Outerは値を出すきっかけになる外側のObservable。Innerは値として流れてくる内側のObservable。

**Flattening**
Higher-order Observableの入れ子を平らにし、Innerの値を1本の流れに乗せること。`mergeMap`・`concatMap`・`switchMap`・`exhaustMap`が担う。

## その他

**Teardown Logic**
購読が解除されたときに実行される後片付けの処理。タイマーの停止やイベントリスナーの解除など。

**Scheduler**
通知をいつ実行するかを制御する仕組み。テストでは仮想時間を使うTestSchedulerが登場する。

**Marble Diagram**
値が時間とともにどう流れるかを表す図。時間を左から右への線とし、その上に値・完了・エラーを置く。

以上が、本書で扱った主な用語です。用語の意味を出発点に、本文へ戻って動きを確かめると、理解がより確かになります。RxJSの学習が、ここから実務へつながっていくことを願っています。
