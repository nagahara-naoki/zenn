---
title: "値を変換・選別・蓄積する"
---

前章で、Operatorをつなぐ`pipe`と、Marble Diagramの読み方を身につけました。準備が整ったので、ここからは具体的なOperatorを扱います。

この章で扱うのは、流れてくる値そのものを扱うOperatorです。値を変換する`map`、副作用を扱う`tap`、値を絞り込む`filter`、連続する重複を除く`distinctUntilChanged`、そして値を蓄積する`scan`と`reduce`です。

どれも使用頻度が高く、RxJSの基礎体力になるOperatorです。1つずつ、Marble Diagramとコードで、動きを確かめていきましょう。

## mapで値を変換する

`map`は、流れてくる値を1つずつ変換するOperatorです。配列の`map`とまったく同じ発想なので、なじみやすいはずです。値の中身を、別の形に変えます。

```text
入力:  --1--2--3--|
           map(n => n * 10)
出力:  --10-20-30-|
```

配列の`map`が要素を1つずつ変換するように、Observableの`map`も値を1つずつ変換します。違いは、対象が「空間に並んだ配列」か、「時間に流れるストリーム」か、それだけです。

実務でいちばんよく使うのは、APIレスポンスの整形です。サーバーから受け取ったデータから、画面が必要とする形だけを取り出します。

```ts
import { of, map } from 'rxjs';

type ApiTask = { id: string; title: string; done: 0 | 1 };

of<ApiTask>({ id: 't1', title: 'RxJSを学ぶ', done: 0 })
  .pipe(
    map((task) => ({
      id: task.id,
      title: task.title,
      completed: task.done === 1, // 0/1をbooleanへ変換
    })),
  )
  .subscribe((task) => console.log(task));

// 出力:
// { id: 't1', title: 'RxJSを学ぶ', completed: false }
```

サーバーが`done: 0 | 1`という形で返してきたものを、アプリで使いやすい`completed: boolean`へ変換しています。

## mapの中で副作用を起こさない

`map`を使うときの注意点があります。`map`が担うのは、値の変換だけです。だから、それ以外のこと、たとえばログ出力や外部の変数の書き換えは書かないようにします。こうした「変換以外のこと」を、副作用と呼びます。

```ts
// 良くない例: mapの中でログを出している（副作用）
map((task) => {
  console.log(task); // ここでは書かない
  return task.title;
});
```

役割を混ぜると、コードの意図が読みにくくなります。`map`は入力から出力への変換に限定し、ログのような副作用は、次に見る`tap`へ分けます。

## mapの中でPromiseを返した場合

もう1つ、初学者がはまりやすい点があります。`map`の中でPromiseやObservableを返しても、`map`はそれを待ってくれません。返したPromiseが、そのまま値として流れてしまいます。

```ts
import { of, map } from 'rxjs';

of('t1')
  .pipe(map((id) => fetch(`/api/tasks/${id}`))) // Promiseがそのまま流れる
  .subscribe((value) => console.log(value)); // value は Promise オブジェクト
```

`subscribe`で受け取れるのは、通信の結果ではなく、Promiseオブジェクトそのものです。これは、`map`が「値をそのまま変換するだけ」で、非同期処理の完了を待つ機能を持たないからです。PromiseやObservableの結果を流したいときは、`mergeMap`や`switchMap`といったFlattening Operatorを使います。この重要なテーマは、「Higher-order ObservableとNested Subscribe」と「Flattening Operator」の章で、じっくり扱います。ここでは、「`map`は待たない」とだけ覚えておいてください。

## tapで副作用を扱う

副作用を実行したいときに使うのが`tap`です。`tap`は、値を変えずに副作用だけを実行するOperatorです。流れてくる値をのぞき見して、ログを出したり、デバッグしたりします。値は、そのまま下流へ通します。

```text
入力:  --1--2--3--|
           tap(n => console.log(n))
出力:  --1--2--3--|   （値は変わらない）
```

```ts
import { of, tap, map } from 'rxjs';

of(1, 2, 3)
  .pipe(
    tap((n) => console.log('変換前:', n)),
    map((n) => n * 10),
    tap((n) => console.log('変換後:', n)),
  )
  .subscribe();

// 出力:
// 変換前: 1
// 変換後: 10
// 変換前: 2
// 変換後: 20
// 変換前: 3
// 変換後: 30
```

`map`との違いは、値を変えるかどうかです。`map`は値を変換して次へ渡し、`tap`は値をそのまま次へ通します。`tap`の中で値を加工したり、値を返したりしても、下流に流れる値は変わりません。あくまで「横からのぞく」だけです。

`tap`は便利ですが、重要な処理を詰め込みすぎないようにします。状態を変える処理を`tap`だらけにすると、流れが追いにくくなります。`tap`は「流れを横から観察する」ための道具、と位置づけてください。

## filterで値を絞り込む

`filter`は、条件に合う値だけを通すOperatorです。条件に合わない値は、下流に流れません。これも配列の`filter`と同じ発想です。

```text
入力:  --1--2--3--4--|
           filter(n => n % 2 === 0)
出力:  -----2-----4--|
```

TypeScriptでは、`filter`を型の絞り込み（Type Guard）にも使えます。少し発展的ですが、便利なので紹介します。戻り値の型を`value is 型`と書くと、通ったあとの値の型が狭まります。

```ts
import { of, filter } from 'rxjs';

of<string | null>('a', null, 'b')
  .pipe(filter((value): value is string => value !== null))
  .subscribe((value) => {
    // ここでの value は string 型（null が除かれている）
    console.log(value.toUpperCase());
  });

// 出力:
// A
// B
```

`null`を取り除きつつ、通ったあとの型を`string`に狭められます。`null`を除いたので、`value.toUpperCase()`を安心して呼べます。値の選別と型の絞り込みを、同時にできるわけです。

## distinctUntilChangedで連続する重複を除く

連続する重複を捨てるのが`distinctUntilChanged`です。名前は長いですが、動きはシンプルで、直前と同じ値が続いたとき、後の値を捨てます。

```text
入力:  --1--1--2--2--1--|
           distinctUntilChanged()
出力:  --1-----2-----1--|
```

最初の`1`は流れ、続く`1`は「直前と同じ」なので捨てられます。`2`に変わると流れ、また`1`に戻ると、それは直前（`2`）と違うので流れます。ここで見てほしいのは、除かれるのが「連続する」重複だけであることです。最後の`1`は、すべての重複を除くなら消えるはずですが、直前が`2`なので残ります。

既定では、値を`===`で比較します。ここで注意です。オブジェクトの場合、`===`は「中身が同じか」ではなく「まったく同じオブジェクトか（参照が同じか）」を比べます。だから、中身が同じでも別のオブジェクトなら「違う値」とみなされます。中身で比べたいときは、比較関数を渡します。

```ts
import { of, distinctUntilChanged } from 'rxjs';

of({ id: 1 }, { id: 1 }, { id: 2 })
  .pipe(distinctUntilChanged((prev, curr) => prev.id === curr.id))
  .subscribe((value) => console.log(value));

// 出力:
// { id: 1 }
// { id: 2 }
```

このOperatorは、検索フォームで効果を発揮します。同じキーワードが連続で届いても、実際に変わったときだけ次へ流せるので、無駄な処理を減らせます。「件数と時間を制御する」の章で、`debounceTime`と組み合わせて使います。

## scanで値を蓄積する

`scan`を使うと、流れてくる値を蓄積できます。前回までの結果と、新しい値を受け取り、新しい結果を返します。そして、値が届くたびに、その時点の結果を流します。

配列の`reduce`に似ていますが、`scan`は「途中の結果を毎回流す」ところが違います。

```text
入力:  --1--2--3--|
           scan((acc, n) => acc + n, 0)
出力:  --1--3--6--|   （1, 1+2, 1+2+3）
```

```ts
import { of, scan } from 'rxjs';

of(1, 2, 3)
  .pipe(scan((total, n) => total + n, 0))
  .subscribe((value) => console.log(value));

// 出力:
// 1
// 3
// 6
```

第2引数の`0`が初期値です。値が届くたびに合計が更新され、その時点の合計が流れます。1が来たら1、2が来たら1+2で3、3が来たら1+2+3で6、という具合です。カウンターや、少しずつ積み上がっていく状態を表すのに向いています。

## reduceで完了時に集計する

`reduce`も、蓄積の考え方は`scan`と同じです。違うのは、結果を流すタイミングです。`reduce`は、`complete`したときに、最後の結果を一度だけ流します。途中の結果は流しません。

```text
入力:  --1--2--3--|
           reduce((acc, n) => acc + n, 0)
出力:  -----------6|   （完了時に最終結果だけ）
```

```ts
import { of, reduce } from 'rxjs';

of(1, 2, 3)
  .pipe(reduce((total, n) => total + n, 0))
  .subscribe((value) => console.log(value));

// 出力:
// 6
```

使い分けはシンプルです。途中経過が必要なら`scan`、最終結果だけでよいなら`reduce`です。1つ注意点があります。`reduce`は`complete`を待つので、終わらないストリームには使えません。永遠に`complete`が来ないと、結果も永遠に流れないからです。この点は、`scan`との使い分けで意識してください。

## イミュータブルな状態更新

`scan`で状態を蓄積するときは、前の状態を書き換えず、新しいオブジェクトを作って返します。この書き方を、イミュータブルな更新と呼びます。

```ts
import { of, scan } from 'rxjs';

type Action = { type: 'add'; value: number };

of<Action>({ type: 'add', value: 1 }, { type: 'add', value: 2 })
  .pipe(
    scan(
      (state, action) => ({ count: state.count + action.value }), // 新しいオブジェクトを返す
      { count: 0 },
    ),
  )
  .subscribe((state) => console.log(state));

// 出力:
// { count: 1 }
// { count: 3 }
```

前の状態`state`を直接書き換えず、`{ count: ... }`という新しいオブジェクトを毎回返しています。だから出力は`{ count: 1 }`、`{ count: 3 }`と、更新のたびに別のオブジェクトになります。この更新方法は、後で扱うSubjectやNgRxの状態管理にも共通します。

## 値を変える・通す・蓄えるでOperatorを選ぶ

値を変換・選別・蓄積するOperatorの違いを整理します。

- `map`は値を1つずつ変換します。副作用は書かず、変換に徹します。
- `map`の中でPromiseを返しても待ってくれません。Flattening Operatorが必要です。
- `tap`は値を変えずに副作用を実行します。ログやデバッグに使います。
- `filter`は条件に合う値だけを通し、TypeScriptではType Guardにも使えます。
- `distinctUntilChanged`は連続する重複を除きます。オブジェクトは比較関数で中身を比べます。
- `scan`は途中結果を毎回流し、`reduce`は完了時に最終結果だけを流します。

次章では、値の件数と時間を制御するOperatorを扱います。`take`や`debounceTime`など、いくつ受け取るか、いつ受け取るかを操るOperatorです。
