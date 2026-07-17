---
title: "はじめてのRxJS"
---

前章で、RxJSの登場人物と処理の流れを、地図として確認しました。

この章では、その地図をたどりながら、いよいよ実際にコードを書きます。RxJSをインストールし、最初のObservableを作り、購読して値を受け取り、Operatorで変換し、最後に購読を解除する。ここまでを、一続きで体験します。

ここで書くコードは、どれも短いものばかりです。細かい仕組みは次章以降でじっくり説明するので、この章では「作って、つないで、購読する」という流れそのものに、手をなじませることを目標にします。頭で理解した地図を、実際に歩いてみる章だと考えてください。

## RxJSをインストールする

RxJSは、npmで公開されているライブラリです。プロジェクトのディレクトリで、インストールします。

```bash
npm install rxjs
```

本書では、TypeScriptでコードを書きます。手元で試すなら、`tsx`のようなツールを使うと、TypeScriptファイルをそのまま実行できて手軽です。

```bash
npm install --save-dev tsx
```

`src/main.ts`のようなファイルを作り、次のコマンドで実行します。

```bash
npx tsx src/main.ts
```

RxJSの機能は、すべて`rxjs`パッケージからimportします。ストリームを作るCreation Functionも、変換するOperatorも、同じ場所から取り出します。この点は覚えておいてください。

```ts
import { of, map, filter } from 'rxjs';
```

## 最初のObservableを作る

では、最初のObservableを作ってみます。`of`を使います。`of`は、渡した値を順番に流すストリームを作る、Creation Functionです。

```ts:src/main.ts
import { of } from 'rxjs';

const numbers$ = of(1, 2, 3);
```

変数名の末尾に`$`を付けると、それがObservableだとひと目でわかります。この`numbers$`は、1、2、3を順に流して完了する、ストリームの設計図です。

ここで、前章で強調した大事な点を、もう一度思い出してください。この時点では、まだ何も起きていません。Observableは設計図にすぎず、購読するまで値は流れません。試しに、この状態のままファイルを実行しても、コンソールには何も表示されないのです。設計図を書いただけでは、料理は始まらない、というわけです。

## subscribeで値を受け取る

値を受け取るには、`subscribe`で購読します。ここで、はじめてストリームが動き出します。

```ts:src/main.ts
import { of } from 'rxjs';

const numbers$ = of(1, 2, 3);

numbers$.subscribe((value) => {
  console.log(value);
});

// 出力:
// 1
// 2
// 3
```

`subscribe`に渡した関数が、値が流れるたびに呼ばれます。`of(1, 2, 3)`なので、1、2、3の順に、3回呼ばれます。

いまは`next`（値の通知）だけを受け取りました。前章で見たとおり、ストリームには`error`や`complete`もありました。これらも受け取りたいときは、関数のかわりにObserverオブジェクトを渡します。

```ts:src/main.ts
import { of } from 'rxjs';

of(1, 2, 3).subscribe({
  next: (value) => console.log('next:', value),
  error: (error) => console.error('error:', error),
  complete: () => console.log('complete'),
});

// 出力:
// next: 1
// next: 2
// next: 3
// complete
```

`of`は、値を流し終えると`complete`します。だから、最後に`complete`が呼ばれています。使い分けの目安はこうです。`next`だけでよいときは関数を、3種類の通知をすべて扱いたいときはObserverオブジェクトを渡します。

## pipeでOperatorをつなぐ

流れてくる値を加工したいときは、Operatorを使います。Operatorは`pipe`でつなぎます。

```ts
observable$.pipe(operator1, operator2).subscribe(observer);
```

`pipe`は、Operatorを左から順に適用し、新しいObservableを返します。ここで、初学者が誤解しやすい点があります。`pipe`は元のObservableを変えるのではなく、変換した「新しいObservable」を返す、ということです。元のObservableはそのまま残ります。この性質のおかげで、1つのObservableから、別々の変換を安心して作れます。詳しくは「Operatorとpipe・Marble Diagramの読み方」の章で扱います。

## mapで値を変換する

`map`は、流れてくる値を1つずつ変換するOperatorです。配列の`map`と同じ発想なので、なじみやすいはずです。

```ts:src/main.ts
import { of, map } from 'rxjs';

of(1, 2, 3)
  .pipe(map((value) => value * 10))
  .subscribe((value) => console.log(value));

// 出力:
// 10
// 20
// 30
```

`map((value) => value * 10)`が、流れてくる値を10倍しています。1は10に、2は20に、3は30に変換されて届きます。値が「変換されて流れる」感覚を、つかんでください。

## filterで値を絞り込む

`filter`は、条件に合う値だけを通すOperatorです。これも配列の`filter`と同じ発想です。

```ts:src/main.ts
import { of, filter } from 'rxjs';

of(1, 2, 3, 4, 5)
  .pipe(filter((value) => value % 2 === 0))
  .subscribe((value) => console.log(value));

// 出力:
// 2
// 4
```

`filter`と`map`は、`pipe`の中で並べてつなげられます。並べた順に、上から適用されます。

```ts:src/main.ts
import { of, filter, map } from 'rxjs';

of(1, 2, 3, 4, 5)
  .pipe(
    filter((value) => value % 2 === 0),
    map((value) => value * 10),
  )
  .subscribe((value) => console.log(value));

// 出力:
// 20
// 40
```

まず`filter`で偶数（2と4）だけを通し、次に`map`でそれを10倍しています。ここで試しに`filter`と`map`の順番を入れ替えると、結果が変わります。Operatorを並べる順番には、意味があるのです。この点も、Operatorの章で詳しく見ます。

## unsubscribeで購読を解除する

`of`は値を流し終えると`complete`するので、購読は自然に終わります。しかし、世の中には、終わらないストリームもあります。

`interval`は、指定した間隔ごとに、0から始まる数値を流し続けるCreation Functionです。放っておくと、いつまでも値が流れ続けます。

```ts:src/main.ts
import { interval } from 'rxjs';

const subscription = interval(1000).subscribe((value) => {
  console.log(value);
});

// 出力（1秒ごと）:
// 0
// 1
// 2
// ...（止めるまで続く）
```

このようなストリームは、必要がなくなったら、こちらから購読を解除します。ここで、`subscribe`の戻り値が役立ちます。`subscribe`はSubscription（購読の実体）を返し、その`unsubscribe`を呼ぶと、購読が終わります。

```ts:src/main.ts
import { interval } from 'rxjs';

const subscription = interval(1000).subscribe((value) => {
  console.log(value);
});

// 3.5秒後に購読を解除する
setTimeout(() => {
  subscription.unsubscribe();
  console.log('購読を解除しました');
}, 3500);

// 出力:
// 0
// 1
// 2
// 購読を解除しました
```

`unsubscribe`を呼ぶと、`interval`は値を流すのをやめます。もし購読を解除しないと、画面から離れても`interval`が裏で動き続け、メモリリークの原因になります。購読の解除は、RxJSでとても重要なテーマなので、「Subscription・購読解除・Observableの自作」の章で、あらためて詳しく扱います。

## 最初のRxJSプログラムを作る

最後に、ここまでの部品を全部組み合わせて、小さなプログラムを作ってみましょう。「1秒ごとに増える数値のうち、偶数だけを10倍して表示し、しばらくしたら購読を解除する」というものです。

```ts:src/main.ts
import { interval, filter, map } from 'rxjs';

const result$ = interval(1000).pipe(
  filter((value) => value % 2 === 0),
  map((value) => value * 10),
);

const subscription = result$.subscribe((value) => {
  console.log(value);
});

setTimeout(() => {
  subscription.unsubscribe();
  console.log('終了');
}, 5500);

// 出力:
// 0    （0秒付近: 0は偶数、0 * 10 = 0）
// 20   （2秒付近: 2は偶数、2 * 10 = 20）
// 40   （4秒付近: 4は偶数、4 * 10 = 40）
// 終了
```

流れを追ってみてください。`interval`でストリームを作り、`filter`で偶数だけを通し、`map`で10倍し、`subscribe`で受け取り、`unsubscribe`で終える。前章で見た「作る→変換する→購読する→受け取る→終わる」の流れが、そっくりそのままコードになっています。

このプログラムは短いですが、RxJSの基本の形が、すべて詰まっています。次章からは、このプログラムに登場した一つひとつの部品が、内部でどう動いているのかを、順番に見ていきます。

## まとめ

この章では、RxJSの基本の流れを、コードで体験しました。

- RxJSは`npm install rxjs`で導入し、機能はすべて`rxjs`からimportします。
- `of`のようなCreation FunctionでObservableを作ります。
- Observableは購読するまで動きません。`subscribe`ではじめて値が流れ始めます。
- `subscribe`には関数かObserverオブジェクトを渡します。3種類の通知を扱うときはObserverオブジェクトを使います。
- `pipe`でOperatorをつなぎ、`map`で変換し、`filter`で絞り込みます。並べる順番には意味があります。
- 終わらないストリームは、`unsubscribe`で購読を解除します。

次章では、いちばんの土台であるObservableの仕組みに戻ります。Observable、Observer、そして`subscribe`したときに内部で何が起きているのかを、順を追って見ていきます。
