---
title: "第29章 signal()・computed()・effect()による状態管理"
---

ここまで、変更検知の仕組みと、それを支えてきたZone.jsを見てきました。この章では、モダンAngularの状態管理の中心にあるSignalsを、体系的に学びます。これまで`signal()`を部分的に使ってきましたが、ここで`signal()`・`computed()`・`effect()`という3つの柱を、その関係とともに整理します。

Signalsは、Angular 20（2025年）で安定版になった、リアクティブな状態管理の仕組みです。「リアクティブ」とは、値の変化に、それに依存する処理が自動で反応することを指します。Signalの最大の特長は、値の変化と、その変化がどこに影響するかを、Angularが正確に追跡できる点です。この追跡こそが、前章までで見た変更検知の効率化と、次章のZonelessを可能にします。3本柱を理解すれば、モダンAngularの状態管理の全体像がつかめます。

:::message
**この章で学ぶこと**

- `signal()`による書き換え可能な状態
- `computed()`による派生状態
- `effect()`による副作用の実行
- `untracked()`と、Signalが変更検知を駆動する仕組み
:::

## signal() — 状態の基本単位

`signal()`は、値を保持する、もっとも基本的なSignalです。第6章から使ってきたとおり、値を関数呼び出しの形で読み取り、`set()`や`update()`で書き換えます。

```ts:src/app/counter.ts
import { Component, signal } from '@angular/core';

@Component({ selector: 'app-counter', template: `<p>{{ count() }}</p>` })
export class Counter {
  protected readonly count = signal(0);

  increment(): void {
    this.count.update((n) => n + 1); // 現在値をもとに更新
  }

  reset(): void {
    this.count.set(0); // 新しい値を直接設定
  }
}
```

`signal(0)`は、初期値`0`を持つSignalを作ります。`count()`で現在の値を読み、`set()`で値を置き換え、`update()`で現在値をもとに新しい値を計算します。数を1増やすように、いまの値を使う更新には`update()`が簡潔です。

Signalが特別なのは、「読まれた場所」をAngularが記録することです。テンプレートで`count()`と読めば、Angularは「このテンプレートは`count`に依存している」と知ります。だからこそ、`count`が変わったときに、そのテンプレートだけを的確に更新できるのです。ただの変数ではなく、変化を通知できる値、それがSignalです。

## computed() — 派生する状態

`computed()`は、ほかのSignalから計算して求まる、派生的なSignalを作ります。第18章で入力から派生値を作るのに使ったものです。

```ts:src/app/cart.ts
import { Component, computed, signal } from '@angular/core';

@Component({ selector: 'app-cart', template: `<p>合計: {{ total() }}円</p>` })
export class Cart {
  protected readonly price = signal(500);
  protected readonly quantity = signal(2);
  protected readonly total = computed(() => this.price() * this.quantity());
}
```

`total`は、`price`と`quantity`から計算されます。どちらかが変われば、`total`も自動で新しい値になります。`computed()`には、重要な性質が3つあります。

- **遅延評価**: `computed()`は、値が読まれるまで計算しません。誰も`total()`を読まなければ、計算は行われません。
- **メモ化**: 一度計算した値は記憶され、依存するSignalが変わるまで再計算されません。同じ値を何度読んでも、計算は一度きりです。
- **読み取り専用**: `computed()`の値は`set()`できません。あくまで、ほかのSignalから導かれる結果だからです。

さらに、依存関係は「実際に読まれたSignal」だけが対象になります。`computed()`の中で条件分岐があり、あるSignalがその時点で読まれなければ、そのSignalへの依存は生じません。この動的な依存追跡により、必要最小限の再計算だけが行われます。派生値は`computed()`で表す、というのがモダンAngularの基本姿勢です。

## effect() — 副作用を実行する

`signal()`と`computed()`が「値」を扱うのに対し、`effect()`は「処理」を扱います。依存するSignalが変わったときに、決めた処理を自動で実行する仕組みです。ここでいう処理とは、ログの出力、外部への保存、DOMやブラウザAPIの操作といった、値を返すのではなく何らかの作用を起こすもの、すなわち副作用を指します。

```ts:src/app/theme.ts
import { Component, effect, signal } from '@angular/core';

@Component({ selector: 'app-theme', template: `...` })
export class Theme {
  protected readonly dark = signal(false);

  constructor() {
    effect(() => {
      // dark が変わるたびに実行される
      document.body.classList.toggle('dark', this.dark());
    });
  }
}
```

この`effect()`は、`dark`が変わるたびに、`body`のクラスを切り替えます。`effect()`の中で読まれたSignal（ここでは`dark`）が、その`effect()`の依存になります。依存が変わると、処理が再実行されます。`effect()`は、注入コンテキスト（多くはコンストラクター内）で登録します。

`effect()`には、後始末の仕組みもあります。コールバックが受け取る`onCleanup`に処理を登録すると、次に再実行される直前や、破棄時に呼ばれます。タイマーの停止や、購読の解除に使えます。

```ts
effect((onCleanup) => {
  const id = setInterval(() => console.log(this.count()), 1000);
  onCleanup(() => clearInterval(id)); // 再実行・破棄の前に後始末
});
```

## effect()は控えめに使う

強力な`effect()`ですが、多用は禁物です。値から値を導くだけなら、`effect()`ではなく`computed()`を使うべきです。`effect()`は、あくまで「Angularの外の世界」に作用させるための最終手段と考えます。

たとえば、「`price`から`total`を求めたい」なら`computed()`です。これを`effect()`の中で`this.total.set(...)`と書くのは、遠回りで、間違いのもとになります。`effect()`が向くのは、`computed()`では表せない、ログ出力や外部APIの呼び出し、DOMの直接操作といった副作用に限られます。「値の派生は`computed()`、外界への作用は`effect()`」という切り分けを、原則にしてください。

## untracked() — 依存から外す

`effect()`や`computed()`の中で、あるSignalの値は読みたいが、その変化には反応させたくない、という場面があります。そのときに使うのが`untracked()`です。

```ts
effect(() => {
  // user が変わったときだけ実行したい。count は値を読むだけ
  console.log(`ユーザー: ${this.user()}, カウント: ${untracked(this.count)}`);
});
```

この`effect()`は、`user`の変化には反応しますが、`count`の変化では再実行されません。`untracked()`で包んだSignalは、依存関係の追跡から外れるためです。意図しない再実行を防ぎたいときに役立ちます。

## Signalが変更検知を駆動する

3本柱を押さえたうえで、前章までとのつながりを確認します。SignalがただのリアクティブAPIにとどまらないのは、変更検知と直結しているからです。

テンプレートでSignalを読むと、Angularは「このビューはこのSignalに依存している」と記録します。そのSignalが`set()`や`update()`で変わると、Angularは依存しているビューに「更新が必要だ」と印を付けます。これは、第27章で見たOnPushの`markForCheck()`が、Signalによって自動で行われるようなものです。開発者が明示的に更新を要求しなくても、Signalの変化が、そのまま画面更新の合図になるのです。

この「Signalの変化が変更検知の起点になる」性質は、決定的な意味を持ちます。これまで変更検知の起点はZone.jsでしたが、Signalがあれば、Zone.jsに頼らずとも「いつ・どこを更新すべきか」がわかります。前章で見たZone.jsの課題を、Signalが根本から解消する道筋が、ここに見えてきます。それが、次章のZonelessです。

## よくあるつまずき

- **`computed()`で済むのに`effect()`を使う**: 値から値を導くだけなら`computed()`です。`effect()`の中で別のSignalを`set()`すると、値の流れが追いにくくなり、無限ループの危険もあります。派生は`computed()`に任せます。
- **`()`を付けずにSignalを比較する**: `if (count)`と書くと、値ではなくSignal（関数）そのものを評価してしまい、常に真になります。値を使うときは必ず`count()`と呼び出します。
- **`effect()`の後始末を忘れる**: `effect()`でタイマーや購読を始めたら、`onCleanup`で確実に止めます。放置すると、破棄後も動き続けてしまいます。
- **オブジェクトSignalの中身だけを書き換える**: `signal({...})`の中身を直接書き換えても、変化として通知されないことがあります。`update()`で新しいオブジェクトに差し替えるのが安全です。

## Signalとテンプレートの相性

最後に、Signalがテンプレートといかに自然になじむかを確認します。テンプレートでは、Signalを`count()`と読むだけで、その値の変化が自動的に表示へ反映されます。第16章で触れたPipeやAsyncPipeと違い、Signalの読み取りには特別な記法も購読の管理も要りません。`computed()`で導いた値も、同じように`total()`と読むだけです。状態をSignalで組み立てておけば、テンプレートは「必要な値を読む」だけで済み、更新の仕組みを意識せずに書けます。この素直さが、Signalファーストで設計する大きな動機になります。

## まとめ

- `signal()`は書き換え可能な状態で、`set()`・`update()`で変更し、`()`で読み取ります
- `computed()`は派生状態で、遅延評価・メモ化され、依存が変わったときだけ再計算されます
- `effect()`は副作用の実行で、依存Signalの変化に反応し、`onCleanup`で後始末できます
- 値の派生は`computed()`、外界への作用は`effect()`と切り分けます
- Signalの変化は変更検知の起点となり、Zone.jsに頼らない更新を可能にします

次章では、このSignalを土台に、Zone.jsを取り除いたZonelessへの転換を、新旧比較で学びます。
