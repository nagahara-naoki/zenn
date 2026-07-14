---
title: "第40章 Subject・BehaviorSubject・ReplaySubject"
---

これまで扱ってきたObservableは、値を「流してくる」側でした。私たちは、それを`subscribe`して受け取るだけでした。しかしときには、自分の好きなタイミングで値を「流し込みたい」こともあります。それを可能にするのが、Subject（サブジェクト）です。

Subjectは、Observableでありながら、同時に値を流し込めるという、二重の性質を持ちます。この性質から、複数の購読者へ同じ値を配る「放送局」のような使い方や、Component間で状態を共有する仕組みの土台として使われてきました。この章では、Subjectと、その性質を少しずつ変えた仲間たち（BehaviorSubject・ReplaySubject）を学びます。あわせて、モダンAngularでこれらがSignalとどう関係するのかにも触れます。

:::message
**この章で学ぶこと**

- Subjectの二重の性質
- BehaviorSubjectとReplaySubjectの違い
- Subjectの使いどころ
- SignalとBehaviorSubjectの関係
:::

## Subjectとは

Subjectは、ObservableとObserverの両方の性質を持つ、特別なオブジェクトです。Observableとして`subscribe`できると同時に、Observerとして`next`で値を流し込めます。

```ts
import { Subject } from 'rxjs';

const subject = new Subject<string>();

// 購読する（Observableとして）
subject.subscribe((value) => console.log('受信:', value));

// 値を流し込む（Observerとして）
subject.next('こんにちは');
subject.next('さようなら');
```

`subject.next('こんにちは')`と、外から値を流し込めるのが、通常のObservableとの違いです。`interval`のようなObservableは、値をいつ流すかをObservable自身が決めていました。Subjectは、私たちが好きなタイミングで値を流せます。

もうひとつの重要な性質が、マルチキャストです。ひとつのSubjectを複数の購読者が`subscribe`すると、`next`で流した値は、すべての購読者に同時に届きます。放送局が電波を流し、複数の受信機がそれを受け取るイメージです。この性質から、Subjectは「複数の場所へ、同じ出来事を知らせる」用途に向きます。

## BehaviorSubject

Subjectには、いくつかの派生があります。もっともよく使うのが、BehaviorSubjectです。

通常のSubjectには、弱点があります。値を流した後で購読を始めた購読者は、それ以前に流れた値を受け取れません。放送に途中から参加しても、それまでの内容は聞けない、というわけです。しかし多くの場面では、「いま現在の値」を、後から参加した購読者にも渡したいものです。

BehaviorSubjectは、この課題を解決します。「現在の値」をひとつ保持し、新しく購読した人には、まずその現在の値を即座に渡します。初期値が必須なのも、この性質のためです。

```ts
import { BehaviorSubject } from 'rxjs';

const count = new BehaviorSubject<number>(0); // 初期値0

count.next(5);

// 後から購読しても、現在の値5を受け取れる
count.subscribe((value) => console.log('現在値:', value)); // 5

// 現在の値を、その場で取り出すこともできる
console.log(count.value); // 5
```

BehaviorSubjectは「現在の値を持つストリーム」なので、状態の保持にうってつけです。第22章で触れた「状態を持つService」の実装として、長らくこのBehaviorSubjectが使われてきました。Serviceの中でBehaviorSubjectに状態を保持し、Componentがそれを購読する、という形です。

## ReplaySubjectとAsyncSubject

もうひとつの派生が、ReplaySubjectです。これは、過去に流した値を、指定した個数だけ記憶しておき、新しい購読者に再生（replay）します。

```ts
import { ReplaySubject } from 'rxjs';

const recent = new ReplaySubject<string>(2); // 直近2件を記憶

recent.next('A');
recent.next('B');
recent.next('C');

// 後から購読すると、直近2件（B, C）を受け取れる
recent.subscribe((value) => console.log(value)); // B, C
```

BehaviorSubjectが「現在の1つ」を渡すのに対し、ReplaySubjectは「直近の複数」を渡せます。履歴を少し遡って伝えたいときに使います。

さらに、AsyncSubjectという派生もあります。これは、完了（complete）したときに、最後の値だけを購読者に渡します。使う場面は限られますが、「最終結果だけが必要」なときに用います。まとめると、Subjectの仲間は「いつ、どの値を、購読者に渡すか」の振る舞いが少しずつ違う、と理解すればよいでしょう。実務でよく使うのは`Subject`と`BehaviorSubject`の2つで、残りは「そういうものもある」と知っておけば十分です。とくに状態を扱うなら`BehaviorSubject`、単なる通知なら`Subject`、という選び方が基本になります。

| 種類 | 新しい購読者に渡すもの |
|---|---|
| `Subject` | 購読後に流れた値のみ |
| `BehaviorSubject` | 現在の値（初期値が必須） |
| `ReplaySubject` | 記憶した直近N件 |
| `AsyncSubject` | 完了時の最後の値のみ |

## Subjectの使いどころ

Subjectは、おもに2つの用途で使われてきました。

ひとつは、**Component間のイベント伝達**です。第17章で、親子を越えた通信にはServiceが要ると述べました。Serviceの中にSubjectを持ち、あるComponentが`next`でイベントを流し、別のComponentがそれを購読する、という形で、離れたComponent間の連絡ができます。「通知」を配る用途です。

もうひとつは、**状態の共有**です。BehaviorSubjectに状態を保持し、複数のComponentがその状態を購読して、同じ値を共有します。これが、第10部で扱うStore Serviceの、古典的な実装でした。具体的には、次のような形です。

```ts:src/app/cart.ts（BehaviorSubjectによるStore Service）
@Injectable({ providedIn: 'root' })
export class CartService {
  // 外へは読み取り専用のObservableとして公開する
  private readonly itemsSubject = new BehaviorSubject<Item[]>([]);
  readonly items$ = this.itemsSubject.asObservable();

  add(item: Item): void {
    const current = this.itemsSubject.value;
    this.itemsSubject.next([...current, item]); // 新しい配列に差し替える
  }
}
```

ここで重要なのが、`asObservable()`です。`itemsSubject`そのものを外へ公開すると、どのComponentからでも`next`で値を流し込めてしまい、状態の変更経路がばらばらになります。`asObservable()`で読み取り専用にして公開すれば、状態を変えられるのは`CartService`の`add`のようなメソッドだけに限定できます。変更の窓口を絞ることが、状態を追いやすく保つ鍵です。Componentは`items$`を`async`パイプで購読し、`add`を呼んで変更を依頼します。単方向データフロー（第17章）の考え方が、状態管理にも通じているのがわかります。

## SignalとBehaviorSubjectの関係

ここで、第6部で学んだSignalを思い出してください。「現在の値を持ち、変化を購読者に伝える」というBehaviorSubjectの役割は、Signalの役割とよく似ています。実際、モダンAngularでは、状態の保持という用途において、BehaviorSubjectをSignalで置き換えられる場面が多くあります。

```ts:BehaviorSubjectによる状態（従来）
private readonly count$ = new BehaviorSubject(0);
```

```ts:Signalによる状態（モダン）
private readonly count = signal(0);
```

どちらも「現在の値を持ち、変化を伝える」点で共通します。Signalのほうが、テンプレートでの読み取りが簡潔で、購読解除の心配もありません。そのため、単純な状態の保持なら、Signalが第一の選択肢になりつつあります。

ただし、RxJSが不要になるわけではありません。`debounceTime`や`switchMap`のような、時間的な制御や非同期の合成は、依然としてRxJSの得意分野です。「現在の値の保持」はSignal、「複雑な非同期の流れ」はRxJS、という役割分担が、モダンAngularの姿です。この共存については、次章でさらに深めます。

## よくあるつまずき

- **通常のSubjectで現在値を期待する**: 通常の`Subject`は、購読前に流れた値を渡しません。現在の値が必要なら`BehaviorSubject`を使います。
- **Subjectを無闇に公開する**: Serviceの外へSubjectをそのまま公開すると、どこからでも`next`で値を流せてしまい、流れの出どころが追えなくなります。外へは`asObservable()`で読み取り専用にして渡すのが安全です。
- **何でもSubjectで状態管理する**: 単純な状態の保持は、Signalのほうが簡潔で安全な場面が増えています。まずSignalで足りないかを考えます。
- **`BehaviorSubject`の`value`に頼りすぎる**: `value`で現在値を同期的に取り出せますが、これに頼った命令的なコードが増えると、リアクティブに書く利点が薄れます。購読や`async`パイプ、あるいはSignalで、変化に反応する形を基本にします。

## まとめ

- Subjectは、Observableでありながら`next`で値を流し込める二重の性質を持ちます
- 複数の購読者へ同時に値を配るマルチキャストが特徴です
- BehaviorSubjectは現在の値を保持し、ReplaySubjectは直近N件を再生します
- Subjectは、Component間のイベント伝達や状態の共有に使われてきました
- 単純な状態の保持は、現在ではSignalで置き換えられる場面が多くあります

次章では、RxJSとSignalという2つのリアクティブの仕組みを、実際につなぐ方法を学びます。
