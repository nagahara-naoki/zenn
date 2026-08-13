---
title: "最新値を組み合わせる"
---

ここまでは、1本のストリームを変換したり選別したりしてきました。ここからは、複数のストリームを組み合わせる、合成に入ります。

実際のアプリでは、1本のストリームだけで完結することは、あまりありません。2つの入力欄の値を合わせて検索条件を作る、複数のAPIの結果をまとめて表示する、状態の変化とユーザー操作を組み合わせる。こうした「複数の流れを1つにまとめる」場面は、いくらでもあります。

RxJSには、複数のObservableを組み合わせる方法が、いくつも用意されています。この章では、そのうち「それぞれの最新値を組み合わせる」やり方を扱います。`combineLatest`と`withLatestFrom`、そして両者を助ける`startWith`です。まず、合成の全体像と選ぶ基準を確認してから、個々のOperatorに入ります。

## 複数のObservableを扱う

合成の方法には、いくつかの種類があります。どれを使うか迷わないために、まず「選ぶときの基準」を持っておきましょう。基準は、次の3つの問いです。

- それぞれの最新値を組み合わせたいのか
- すべての完了を待って、結果を集めたいのか
- 到着した順に、1本にまとめたいのか

この章で扱うのは、1つ目の「最新値を組み合わせる」やり方です。2つ目と3つ目は、次章の「完了待ちと到着順」で扱います。まずは、この分類を頭に入れておくと、合成のOperatorが整理しやすくなります。

## Creation FunctionとPipeable Operatorの違い

合成のOperatorには、Creation Functionの形と、Pipeable Operatorの形の、両方があるものがあります。少しややこしいので、先に触れておきます。たとえば`combineLatest`は、Creation Functionとして使います。

```ts
import { combineLatest } from 'rxjs';

combineLatest([a$, b$]).subscribe(([a, b]) => console.log(a, b));
```

同じ組み合わせを、Pipeable Operatorの`combineLatestWith`でも書けます。

```ts
import { combineLatestWith } from 'rxjs';

a$.pipe(combineLatestWith(b$)).subscribe(([a, b]) => console.log(a, b));
```

どちらも結果は同じです。使い分けの目安は、こうです。複数のObservableを対等に並べたいときは、Creation Functionの形（`combineLatest([a$, b$])`）が読みやすくなります。1本のストリームに、`pipe`の途中で合成をつなげたいときは、Pipeable Operatorの形が収まります。本書では、対等に並べる場面が多いので、Creation Functionの形を主に使います。

## combineLatestで最新値を組み合わせる

`combineLatest`は、複数のObservableそれぞれの最新値を組み合わせて流します。動きの要点はこうです。どれか1つでも新しい値を流すと、そのときのすべての最新値を、配列にして流します。

```text
a$:      --1-----3--------5-->
b$:      -----A-------B------->
             combineLatest([a$, b$])
出力:    -----[1,A]-[3,A]-[3,B]-[5,B]->
```

図を追ってみましょう。`a$`が`3`になると、そのときの`b$`の最新値`A`と組み合わさって、`[3, A]`が流れます。次に`b$`が`B`になると、そのときの`a$`の最新値`3`と組み合わさって、`[3, B]`が流れます。どちらのObservableが動いても、その瞬間の「両方の最新値」がセットで流れる、というわけです。

```ts
import { combineLatest, fromEvent, map } from 'rxjs';

const keyword$ = fromEvent<InputEvent>(keywordInput, 'input').pipe(
  map((e) => (e.target as HTMLInputElement).value),
);
const category$ = fromEvent<Event>(categorySelect, 'change').pipe(
  map((e) => (e.target as HTMLSelectElement).value),
);

combineLatest([keyword$, category$]).subscribe(([keyword, category]) => {
  console.log('検索条件:', keyword, category);
});
```

キーワードとカテゴリの、どちらを変えても、その時点の両方の値がそろって届きます。複数の条件を組み合わせて、1つの結果を作りたいときに、うってつけです。

## 初回通知の条件

`combineLatest`には、初学者がはまりやすい性質があります。最初の値を流すのは、すべてのObservableが少なくとも1回、値を流したあとです。

```text
a$:      --1--2-----4-->
b$:      --------A----->
             combineLatest([a$, b$])
出力:    --------[2,A]-[4,A]->
```

`a$`が`1`や`2`を流しても、`b$`がまだ何も流していないあいだは、`combineLatest`は何も流しません。全員がそろうのを待っているのです。`b$`が`A`を流して、はじめて`[2, A]`が流れます。「全員がそろうまで待つ」という動きを、覚えておいてください。

## 値が一度も通知されない場合

この「全員を待つ」性質は、そのまま落とし穴にもなります。組み合わせるObservableのうち、たった1つでも、一度も値を流さないと、`combineLatest`は永遠に何も流しません。

たとえば、ユーザーが操作するまで値を流さない入力欄を組み合わせたとします。その欄が一度も操作されないかぎり、`combineLatest`の結果は出てきません。「なぜか何も表示されない」と悩んだら、まず「どれかのObservableが、まだ初回の値を流していないのでは?」と疑ってみてください。この問題は、次に見る`startWith`で解決できます。

## startWithで初期値を与える

`startWith`は、ストリームの先頭に初期値を差し込むOperatorです。購読した瞬間に、まず指定した値が流れます。

```text
入力:      -----A----->
               startWith('初期')
出力:      初期-----A----->
```

これを`combineLatest`と組み合わせると、さきほどの初回通知の問題を、きれいに解決できます。それぞれのObservableに初期値を与えておけば、購読直後から値がそろい、すぐに結果が流れ始めるからです。

```ts
import { combineLatest, fromEvent, map, startWith } from 'rxjs';

const keyword$ = fromEvent<InputEvent>(keywordInput, 'input').pipe(
  map((e) => (e.target as HTMLInputElement).value),
  startWith(''), // 初期値として空文字を流す
);
const category$ = fromEvent<Event>(categorySelect, 'change').pipe(
  map((e) => (e.target as HTMLSelectElement).value),
  startWith('all'), // 初期カテゴリ
);

combineLatest([keyword$, category$]).subscribe(([keyword, category]) => {
  console.log('検索条件:', keyword, category);
});
// 購読直後に ['', 'all'] が流れる
```

`startWith`で初期値を与えたので、ユーザーがまだ何も操作していなくても、購読直後に`['', 'all']`が流れます。おかげで、初期状態から画面を組み立てられます。`combineLatest`と`startWith`は、セットで使うことが多い組み合わせです。

## withLatestFrom

`withLatestFrom`は、主となるObservableが値を流したときだけ、ほかのObservableの最新値を添えて流すOperatorです。

`combineLatest`との違いは、「どれが流れの引き金になるか」です。`combineLatest`は、どのObservableが動いても流れました。いっぽう`withLatestFrom`は、主となるObservableが動いたときだけ流れます。

```text
main$:   --------X--------Y-->
other$:  --a--b-----c-------->
             main$.pipe(withLatestFrom(other$))
出力:    --------[X,b]----[Y,c]->
```

`other$`がいくら値を流しても、それだけでは何も流れません。`main$`が`X`を流したときに、そのときの`other$`の最新値`b`を添えて、`[X, b]`が流れます。`other$`は、あくまで「参照される補助役」です。主役ではありません。

## フォーム値と状態を組み合わせる

`withLatestFrom`が向くのは、「あるきっかけのときに、別の最新状態を参照したい」場面です。たとえば、送信ボタンが押されたときに、そのときのフォームの状態を読み取る、という処理です。

```ts
import { fromEvent, withLatestFrom, map } from 'rxjs';

const submit$ = fromEvent(submitButton, 'click');
const formValue$ = fromEvent<InputEvent>(input, 'input').pipe(
  map((e) => (e.target as HTMLInputElement).value),
);

submit$
  .pipe(withLatestFrom(formValue$))
  .subscribe(([, value]) => {
    console.log('送信:', value);
  });
```

引き金は、ボタンのクリック（`submit$`）です。クリックされた瞬間の、フォームの最新値（`formValue$`）を読み取ります。フォームに入力しただけでは、何も起きません。「引き金」と「参照」を分けたいときに、`withLatestFrom`がぴたりとはまります。

## combineLatestとwithLatestFromの使い分け

2つの違いを、表にまとめます。

| 観点 | `combineLatest` | `withLatestFrom` |
|---|---|---|
| 流れる引き金 | どのObservableでも | 主となるObservableだけ |
| ほかのObservableの役割 | 対等 | 参照される補助役 |
| 向く場面 | 複数の条件を対等に組み合わせる | きっかけのときに最新状態を参照する |

どのObservableが変わっても結果を更新したいなら`combineLatest`、特定のきっかけのときにだけ他の最新値を参照したいなら`withLatestFrom`です。「全員が引き金か、それとも主役だけが引き金か」で見分けられます。

## すべてが引き金ならcombineLatest、主役があるならwithLatestFrom

最新値を組み合わせる2つの方法を整理します。

- `combineLatest`は、どれかが動いたとき、すべての最新値を組み合わせて流します。
- ただし、全員が最初の値を流すまで、何も流しません。1つでも沈黙すると結果が出ません。
- `startWith`で初期値を与えると、初回通知の問題を解決できます。
- `withLatestFrom`は、主となるObservableが動いたときだけ、ほかの最新値を添えて流します。
- 全員が引き金なら`combineLatest`、主役だけが引き金なら`withLatestFrom`を選びます。

次章では、もう2つの合成を扱います。すべての完了を待つ`forkJoin`と、到着順や順番でまとめる`merge`・`concat`・`zip`・`race`です。
