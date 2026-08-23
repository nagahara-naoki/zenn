---
title: "ColdとHot・同期と非同期"
---

「Observable・Observer・subscribeの仕組み」の章では、同じ`interval`を2回購読すると、購読ごとにタイマーが作られると確認しました。

実は、この性質を持つObservableには名前があります。Cold Observableです。そして、これとは対照的に、実行を共有するHot Observableもあります。この章では、まずColdとHotの違いを、じっくり観察します。

後半では、もう1つのよくある誤解を解きます。「Observableは常に非同期だ」と思われがちですが、そうとは限りません。`of`のように、購読した瞬間に値を流し切ってしまう、同期的なObservableもあるのです。この同期と非同期の違いを、実行順序とイベントループの視点から確かめます。

## Cold Observable

Cold Observableは、購読するたびに、新しい実行が始まるObservableです。値を生み出すProducerが、購読のたびに、あらためて作られます。

そこで見た`interval`が、まさにこれでした。2つの購読は、それぞれ自分のカウントを0から始めます。

```text
購読A: --0--1--2--3-->
購読B:       --0--1--2-->   （Bは自分の0から始まる）
```

Cold Observableでは、購読者ごとにProducerが独立します。身近なものにたとえると、動画のオンデマンド配信に似ています。誰がいつ再生ボタンを押しても、その人のために、動画は最初から再生されます。視聴者どうしは、まったく独立しています。

## Hot Observable

Hot Observableは、購読とは別に存在するProducerを観察するObservableです。Producerは購読者が現れる前から値を生み出せるため、遅れて購読すると過去の値を受け取れません。

DOMのクリックイベントが、その代表です。クリックは、購読者が何人いようと関係なく発生します。複数の購読者は、同じ1回のクリックを、一緒に受け取ります。

```ts
import { fromEvent } from 'rxjs';

const clicks$ = fromEvent<MouseEvent>(document, 'click');

clicks$.subscribe(() => console.log('A: クリック'));
clicks$.subscribe(() => console.log('B: クリック'));

// 1回クリックすると:
// A: クリック
// B: クリック
```

1回のクリックで、AとBの両方が反応しています。ただし、`fromEvent`は購読ごとにイベントリスナーを登録します。クリックという外部Producerは共通でも、1つのRxJS内部実行を共有しているわけではありません。この違いは、後でUnicastとMulticastを分けて考えると見通しがよくなります。

HotなProducerは、テレビの生放送に似ています。放送は、視聴者がいようといまいと進みます。途中からチャンネルを合わせた人は、それより前の場面を見られません。同じように、HotなProducerへ遅れて接続した購読者は、それ以前の値を通常は受け取りません。

```text
Producer: --a--b--c--d--e-->
購読A:    ^（最初から）  a  b  c  d  e
購読B:          ^（cから購読）  c  d  e
```

## ColdとHotの違いを整理する

2つの違いを、表にまとめます。

| 観点 | Cold Observable | Hot Observable |
|---|---|---|
| Producerの位置 | Observableの内側 | Observableの外側 |
| 購読したとき | 新しい実行が始まる | 進行中の実行に相乗りする |
| 値の発生 | 購読ごとに独立 | 購読とは独立したProducerから届く |
| 遅れて購読すると | 最初から受け取る | 途中から受け取る |

見分ける鍵は、Producerがどこにあるかです。Producerが購読のたびに作られるならCold、購読とは別に外で動いているならHotです。オンデマンド配信（Cold）か、生放送（Hot）か、と考えると区別しやすくなります。

## 身近な例を分類する

代表的な非同期処理を、ColdとHotで分類してみましょう。

| 例 | 分類 | 理由 |
|---|---|---|
| `defer(() => from(fetch(...)))` | Cold | 購読するたびに`fetch`を新しく呼ぶ |
| `from(fetch(...))` | すでに開始済み | `fetch`はObservableを作る前に1回だけ始まる |
| `interval`・`timer` | Cold | 購読するたびに新しいタイマーが始まる |
| DOMイベント | HotなProducer | イベントは購読とは無関係に発生する |
| 接続済みのWebSocket | HotなProducer | メッセージは購読とは無関係に届く |

HTTP通信の分類は、HTTPという処理ではなく、Observableへの包み方で決まります。Angularの`HttpClient`、`fromFetch`、`defer(() => from(fetch(...)))`のように購読時にリクエストを作ればColdです。対して`from(fetch(...))`では、先に作られた同じPromiseを観察するため、複数回購読してもリクエストは増えません。「HTTPは常にCold」と覚えず、リクエストがどの時点で作られるかを確認してください。

## UnicastとMulticast

Cold/Hotと一緒に語られやすい言葉が、UnicastとMulticastです。ただし、この2組は同義ではありません。

Unicastは、購読者ごとに専用の通知経路を作ることです。Multicastは、1つの上流購読から届いた通知を複数の購読者へ配ることです。Coldな`interval`は通常Unicastですが、`share`を使えば1つのタイマーをMulticastできます。HotなDOMイベントを`fromEvent`で扱う場合は、購読ごとにリスナーを登録するため、HotなProducerを観察していてもRxJS内部の購読は共有されていません。

```mermaid
flowchart TD
  subgraph Unicast["Unicast（購読ごとに通知経路）"]
    P1["Producer"] --> A1["購読A"]
    P2["Producer"] --> B1["購読B"]
  end
  subgraph Multicast["Multicast（1つの上流購読を共有）"]
    P3["Producer"] --> A2["購読A"]
    P3 --> B2["購読B"]
  end
```

この章では、Producerが購読時に作られるか、購読とは独立して存在するかをCold/Hotで捉えます。通知経路を共有するUnicast/Multicastは別の軸です。Cold Observableへの1つの購読を複数へ配る方法は、「SubjectとMulticast」の章で扱います。

## ColdとHotを二択だけで考えない

ここまで、きれいに2つに分けて説明してきました。しかし、注意してほしいことがあります。現実のObservableは、いつもきれいに二分できるわけではありません。

同じ`interval`でも、Operatorで共有すればHotのように振る舞います。逆に、Hotなイベントも、購読の仕方によっては途中で切り出せます。ColdかHotかは、そのObservableに最初から刻まれた固定の属性ではなく、Producerがどこにあり、どう共有されているかで決まる、動的なものなのです。

ですから、大事なのは、ラベルを貼って安心することではありません。「この購読で、新しい実行が始まるのか。それとも、既存の実行に相乗りするのか」を、そのつど考えられることです。この問いは、多重リクエストやメモリリークを見抜くときに、実際に役立ちます。

## Observableは常に非同期とは限らない

ここから、話題を変えます。ここまでObservableを「非同期処理のための道具」として説明してきましたが、Observableそのものは、必ずしも非同期ではありません。これは、多くの初学者が意外に思う点です。

`of`は、購読したその場で、値を流し切ります。つまり、同期的に動きます。次のコードの出力順を、予想してみてください。

```ts
import { of } from 'rxjs';

console.log('前');
of(1, 2, 3).subscribe((value) => console.log(value));
console.log('後');

// 出力:
// 前
// 1
// 2
// 3
// 後
```

`1`、`2`、`3`は、「後」よりも先に出力されます。`of`は、購読した瞬間に、すべての値を同期的に流し切り、それから次の行へ進むからです。「非同期だから、あとで流れるはず」と予想していると、この順番は意外に見えるでしょう。

## of・from・intervalの実行タイミング

Creation Functionによって、同期か非同期かが分かれます。

| Creation Function | 実行 |
|---|---|
| `of(...)` | 同期 |
| `from(配列)` | 同期 |
| `from(Promise)` | 非同期 |
| `interval` / `timer` | 非同期 |

`of`と、配列からの`from`は、同期です。渡された値はその場にすべてあるので、待つ必要がないからです。一方、`interval`や`timer`は、時間を扱うので非同期です。`from`は、渡すものによって変わります。配列なら同期、`Promise`なら（Promiseの完了を待つので）非同期になります。

## 同期通知の注意点

同期Observableには、気をつけたい点があります。購読のすぐ後ろに書いたコードが、値をすべて受け取ったあとに実行される、という順序です。この順序を忘れると、思わぬバグになります。

```ts
import { of } from 'rxjs';

let result = 0;
of(1, 2, 3).subscribe((value) => {
  result += value;
});
console.log(result); // 6（ofは同期なので、ここでは合計が出そろっている）
```

この例は、うまくいきます。`of`が同期だからです。購読が終わった時点で、`result`にはすでに合計の6が入っています。しかし、もし同じコードを非同期のObservableに変えたら、どうなるでしょうか。`console.log`のほうが先に走り、`result`はまだ`0`のままです。値がまだ流れていないからです。この違いは、初学者がはまりやすい落とし穴です。購読の外で結果を使うときは、そのObservableが同期か非同期かを意識してください。より安全なのは、結果を使う処理も、購読の中に書いてしまうことです。

## JavaScriptのイベントループとの関係

同期と非同期の違いは、JavaScriptのイベントループという仕組みと関係しています。少し発展的な話ですが、順序を予測する助けになります。

同期の`of`は、いま動いているコードと同じタイミングで値を流します。`Promise`からの`from`は、マイクロタスクという扱いで、いまのコードが終わった直後に流します。`setTimeout`を使う`timer`や`interval`は、マクロタスクという扱いで、さらにあとに流します。実行される順番には、この優先順位があるのです。

```ts
import { of, from } from 'rxjs';

console.log('1: 同期');
from(Promise.resolve('3: マイクロタスク')).subscribe((v) => console.log(v));
of('2: 同期').subscribe((v) => console.log(v));

// 出力:
// 1: 同期
// 2: 同期
// 3: マイクロタスク
```

同期の通知（1と2）が先に出そろい、`Promise`由来の通知（3）は、そのあとに回っています。実行タイミングは、値の発生源と、Operatorが利用するSchedulerによって決まります。`from(Promise)`の通知はPromiseのマイクロタスク、`timer`や`interval`は既定の`asyncScheduler`を利用します。「同期のObservableもある」と知っておくと、実行順序で迷ったときに原因を探しやすくなります。

## Producerを共有するかどうかでColdとHotを見分ける

ColdとHot、同期と非同期の違いを整理します。

- Cold Observableは購読ごとに新しい実行が始まり、購読者どうしは独立します（オンデマンド配信のイメージ）。
- Hot Observableは購読とは別に動くProducerを観察し、遅れて購読すると以前の値を通常は受け取りません（生放送のイメージ）。
- 見分ける鍵はProducerの位置です。`interval`は購読ごとにタイマーを作るCold、DOMイベントは購読外で発生するHotです。HTTPはObservableへの包み方を確認します。
- Cold/HotはProducerの開始時点、Unicast/Multicastは通知の共有方法を表す別の軸です。
- Observableは常に非同期とは限りません。`of`や配列からの`from`は同期的に値を流します。
- 実行順序はイベントループと関係し、同期・マイクロタスク・マクロタスクの順に流れます。

次章からは、Observableの作り方を体系的に見ていきます。まず`of`、`from`、`fromEvent`、`interval`、`timer`という、もっともよく使うCreation Functionを扱います。
