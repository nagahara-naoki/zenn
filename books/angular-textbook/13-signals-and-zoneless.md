---
title: "Signalsとzoneless"
---

この章では、モダンAngularの状態管理の中心であるSignalsを体系的に学び、それを土台にZone.jsを取り除くzonelessへの転換を扱います。

:::message
**この章で学ぶこと**

- `signal()`による書き換え可能な状態
- `computed()`による派生状態
- ZonelessとZone.js方式の違い
- Zonelessで変更検知が走る条件
:::

## signal()・computed()・effect()による状態管理

ここまで、変更検知の仕組みと、それを支えてきたZone.jsを見てきました。この節では、モダンAngularの状態管理の中心にあるSignalsを、体系的に学びます。これまで`signal()`を部分的に使ってきましたが、ここで`signal()`・`computed()`・`effect()`という3つの柱を、その関係とともに整理します。

Signalsは、Angular 20（2025年）で安定版になった、リアクティブな状態管理の仕組みです。「リアクティブ」とは、値の変化に、それに依存する処理が自動で反応することを指します。Signalの最大の特長は、値の変化と、その変化がどこに影響するかを、Angularが正確に追跡できる点です。この追跡こそが、前章までで見た変更検知の効率化と、次章のZonelessを可能にします。3本柱を理解すれば、モダンAngularの状態管理の全体像がつかめます。

### signal() — 状態の基本単位

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

### computed() — 派生する状態

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

### effect() — 副作用を実行する

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

### effect()は控えめに使う

強力な`effect()`ですが、多用は禁物です。値から値を導くだけなら、`effect()`ではなく`computed()`を使うべきです。`effect()`は、あくまで「Angularの外の世界」に作用させるための最終手段と考えます。

たとえば、「`price`から`total`を求めたい」なら`computed()`です。これを`effect()`の中で`this.total.set(...)`と書くのは、遠回りで、間違いのもとになります。`effect()`が向くのは、`computed()`では表せない、ログ出力や外部APIの呼び出し、DOMの直接操作といった副作用に限られます。「値の派生は`computed()`、外界への作用は`effect()`」という切り分けを、原則にしてください。

### untracked() — 依存から外す

`effect()`や`computed()`の中で、あるSignalの値は読みたいが、その変化には反応させたくない、という場面があります。そのときに使うのが`untracked()`です。

```ts
effect(() => {
  // user が変わったときだけ実行したい。count は値を読むだけ
  console.log(`ユーザー: ${this.user()}, カウント: ${untracked(this.count)}`);
});
```

この`effect()`は、`user`の変化には反応しますが、`count`の変化では再実行されません。`untracked()`で包んだSignalは、依存関係の追跡から外れるためです。意図しない再実行を防ぎたいときに役立ちます。

### Signalが変更検知を駆動する

3本柱を押さえたうえで、前章までとのつながりを確認します。SignalがただのリアクティブAPIにとどまらないのは、変更検知と直結しているからです。

テンプレートでSignalを読むと、Angularは「このビューはこのSignalに依存している」と記録します。そのSignalが`set()`や`update()`で変わると、Angularは依存しているビューに「更新が必要だ」と印を付けます。これは、第27章で見たOnPushの`markForCheck()`が、Signalによって自動で行われるようなものです。開発者が明示的に更新を要求しなくても、Signalの変化が、そのまま画面更新の合図になるのです。

この「Signalの変化が変更検知の起点になる」性質は、決定的な意味を持ちます。これまで変更検知の起点はZone.jsでしたが、Signalがあれば、Zone.jsに頼らずとも「いつ・どこを更新すべきか」がわかります。前章で見たZone.jsの課題を、Signalが根本から解消する道筋が、ここに見えてきます。それが、次章のZonelessです。

### よくあるつまずき

- **`computed()`で済むのに`effect()`を使う**: 値から値を導くだけなら`computed()`です。`effect()`の中で別のSignalを`set()`すると、値の流れが追いにくくなり、無限ループの危険もあります。派生は`computed()`に任せます。
- **`()`を付けずにSignalを比較する**: `if (count)`と書くと、値ではなくSignal（関数）そのものを評価してしまい、常に真になります。値を使うときは必ず`count()`と呼び出します。
- **`effect()`の後始末を忘れる**: `effect()`でタイマーや購読を始めたら、`onCleanup`で確実に止めます。放置すると、破棄後も動き続けてしまいます。
- **オブジェクトSignalの中身だけを書き換える**: `signal({...})`の中身を直接書き換えても、変化として通知されないことがあります。`update()`で新しいオブジェクトに差し替えるのが安全です。

### Signalとテンプレートの相性

最後に、Signalがテンプレートといかに自然になじむかを確認します。テンプレートでは、Signalを`count()`と読むだけで、その値の変化が自動的に表示へ反映されます。第16章で触れたPipeやAsyncPipeと違い、Signalの読み取りには特別な記法も購読の管理も要りません。`computed()`で導いた値も、同じように`total()`と読むだけです。状態をSignalで組み立てておけば、テンプレートは「必要な値を読む」だけで済み、更新の仕組みを意識せずに書けます。この素直さが、Signalファーストで設計する大きな動機になります。

## NgZoneからSignals・Zonelessへ

第6部の締めくくりとして、変更検知の大きな転換点であるZonelessを学びます。第28章で、Zone.jsが変更検知を起動していたこと、そしてその課題を見ました。第29章では、Signalが変更検知の起点になれることを見ました。この2つを結ぶと、ひとつの結論が導かれます。「Signalがあれば、Zone.jsは要らないのではないか」。この発想を実現したのが、Zonelessです。

Zonelessは、Zone.jsを取り除いてアプリケーションを動かす方式です。実験的な導入を経て、Angular 20.2で安定版となり、Angular 21（2025年）では新規プロジェクトの既定になりました。本書が基準とするv22世代では、Zonelessが標準的な選択肢であり、これから学ぶ書き方も、すべてZonelessを前提として問題なく動きます。この節では、Zone.js方式とZoneless方式を比較し、何が変わり、開発者は何を意識すればよいのかを整理します。

### Zonelessとは何か

Zonelessとは、その名のとおり「ゾーン（Zone.js）がない」状態でAngularを動かすことです。第28章で見たように、従来はZone.jsが非同期の出来事を監視し、それを合図に変更検知を起動していました。Zonelessでは、この監視役がいません。代わりに、「状態が変わった」という明示的な通知を受けて、変更検知が走ります。

考え方の違いを、ひとことで表せます。Zone.js方式は「何か非同期が起きた。念のため確認しよう」という方式でした。Zoneless方式は「ここが変わった、と知らされた。そこを更新しよう」という方式です。前者は起こりうる変化を広く拾い、後者は実際の変化を正確に受け取ります。この違いが、効率の差を生みます。

### Zonelessで変更検知が走る条件

Zone.jsがいないと、Angularはどうやって「状態が変わった」ことを知るのでしょうか。Zonelessでは、次のような明示的な通知が、変更検知の起点になります。

- **Signalの更新**: テンプレートで読まれているSignalが変わると、Angularに通知されます
- **`AsyncPipe`の発火**: `async`パイプが新しい値を受け取ると、更新が要求されます
- **イベントリスナー**: テンプレートやホストのイベントハンドラーが実行されると、検知が走ります
- **`markForCheck()`の呼び出し**: `ChangeDetectorRef`で明示的に更新を要求した場合
- **`setInput()`**: 動的に生成したComponentへ入力を設定した場合

見比べてわかるとおり、これらは第27章で挙げたOnPushの更新条件と、ほぼ重なります。実は、Zonelessは「アプリ全体がOnPushで動く」ようなものだと考えられます。だからこそ、前節までで学んだOnPushとSignalの組み合わせが、そのままZonelessの土台になるのです。

とりわけ重要なのがSignalです。Signalで状態を持っていれば、その変化は自動でAngularに通知されます。開発者が`markForCheck()`を呼ぶ必要はありません。Zonelessで快適に開発する鍵は、状態をSignalで管理することにあります。

### Zonelessを有効にする

Zonelessは、アプリケーションの起動設定で有効にします。第4章で見た`app.config.ts`に、`provideZonelessChangeDetection()`を加えます。

```ts:src/app/app.config.ts
import { ApplicationConfig, provideZonelessChangeDetection } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [provideZonelessChangeDetection()],
};
```

Angular 21以降の新規プロジェクトでは、これが既定で設定されているため、自分で書く必要はありません。以前のバージョンや、Zone.jsを使っていた既存プロジェクトをZoneless化する場合に、この指定を加えます。

あわせて、不要になったZone.js本体を取り除きます。`angular.json`のpolyfills設定から`zone.js`の記述を削除し、パッケージ自体もアンインストールします。

```bash
npm uninstall zone.js
```

これで、Zone.jsを読み込まない、正真正銘のZonelessアプリケーションになります。ただし、既存プロジェクトの場合は、Zone.jsの自動起動に頼っていた箇所がないかを確認してから取り除くのが安全です。

### 新旧を比べる

Zone.js方式とZoneless方式の違いを、表に整理します。

| 観点 | Zone.js方式（旧） | Zoneless方式（新） |
|---|---|---|
| 変更検知の起点 | 非同期の出来事をZone.jsが検知 | Signalなどの明示的な通知 |
| 検知の範囲 | 広く（起きうる変化を拾う） | 狭く（実際の変化を受け取る） |
| バンドルサイズ | Zone.jsの分だけ大きい | Zone.jsがなく小さい |
| デバッグ | スタックトレースが複雑 | 素直なスタックトレース |
| 状態管理の前提 | 任意（プロパティでも動く） | Signalが基本 |

Zonelessは、Zone.jsの4つの課題――過剰な変更検知、バンドルサイズ、デバッグの難しさ、エコシステムとの相性――を、まとめて解消します。Zone.jsという監視の層がなくなることで、アプリは軽く、速く、追いやすくなります。

### Zonelessへの移行の指針

これから新しくアプリを作るなら、Zonelessが既定であり、特別な準備は要りません。状態をSignalで持ち、非同期の表示には`async`パイプを使う、という本書で学んできた書き方が、そのままZonelessに適合します。

一方、Zone.jsに依存した既存プロジェクトを移行する場合は、いくつか確認すべき点があります。たとえば、`setTimeout`のコールバックの中でプロパティを書き換え、Zone.jsによる自動検知に頼って画面を更新していた箇所は、Zonelessでは更新されないことがあります。こうした箇所は、状態をSignalに置き換えるのが、もっとも素直な対処です。第28章で触れた`runOutsideAngular`のような、Zone.jsを前提とした最適化も、Zonelessでは不要になり、書き換えの対象になります。

移行は一度に済ませる必要はありません。まずOnPushとSignalへ寄せてアプリを整え、Zone.jsへの依存を減らしてから、Zonelessに切り替える、という段階的な進め方が現実的です。本書がSignalファーストで書いてきたのは、この移行のしやすさも見据えてのことでした。

### 新旧のコードを比べる

Zone.jsに依存したコードが、Zonelessでどう書き換わるのかを、具体例で見ます。次は、1秒ごとに残り時間を減らすカウントダウンです。従来は、Zone.jsの自動検知に頼って、ふつうのプロパティを書き換えていました。

```ts:旧来の書き方（Zone.jsの自動検知に依存）
export class Countdown {
  remaining = 60;

  start(): void {
    setInterval(() => {
      this.remaining--; // Zone.jsが検知して画面を更新していた
    }, 1000);
  }
}
```

このコードは、Zone.jsがあれば動きますが、Zonelessでは`remaining`の変化が通知されず、画面が更新されません。Zonelessに適合させるには、状態をSignalにします。

```ts:src/app/countdown.ts（現在の書き方）
export class Countdown {
  protected readonly remaining = signal(60);

  start(): void {
    setInterval(() => {
      this.remaining.update((n) => n - 1); // Signalの変化が通知される
    }, 1000);
  }
}
```

変えたのは、プロパティをSignalにし、`this.remaining--`を`this.remaining.update(...)`にしただけです。これで、Zone.jsの有無にかかわらず、確実に画面が更新されます。Signalが変化を明示的に通知するため、暗黙の監視に頼る必要がなくなったのです。この書き換えの容易さが、Signalファーストで書いておくことの利点です。

### ハイブリッドな移行

大規模な既存アプリを、一度に完全なZonelessへ移すのは容易ではありません。そのため、段階的な移行の道も用意されています。Zone.jsを残したまま、Signalによる効率的な変更検知の恩恵を先に受け、準備が整った部分からZone.jsへの依存を外していく、という進め方です。

現実的な手順としては、次のようになります。まず、状態をSignalへ、非同期の表示を`async`パイプへ置き換えます。次に、`setTimeout`のコールバック内での直接的なプロパティ書き換えなど、Zone.jsの自動検知に暗黙的に頼った箇所を洗い出し、Signalベースに直します。最後に、Zone.jsを取り除いてZonelessに切り替え、動作を確認します。焦らず、一部分ずつ検証しながら進めるのが安全です。

:::message alert
既存アプリからZone.jsを取り除く前に、Zone.jsの自動検知に依存した箇所が残っていないかを十分に確認してください。見落とすと、特定の操作で画面が更新されない不具合につながります。状態をSignalへ移す作業を先に済ませてから、Zone.jsの除去に着手するのが安全です。
:::

:::message
Zonelessは、Signalで状態を管理していれば、多くの場合そのまま動きます。逆に、Zone.jsの自動検知に暗黙的に頼ったコードは、移行時に見直しが必要です。日ごろからSignalファーストで書いておくことが、Zoneless時代への最良の備えになります。
:::

## まとめ

- `signal()`は書き換え可能な状態で、`set()`・`update()`で変更し、`()`で読み取ります
- `computed()`は派生状態で、遅延評価・メモ化され、依存が変わったときだけ再計算されます
- `effect()`は副作用の実行で、依存Signalの変化に反応し、`onCleanup`で後始末できます
- Zonelessは、Zone.jsを取り除き、明示的な通知で変更検知を走らせる方式です
- 起点はSignalの更新・`async`パイプ・イベント・`markForCheck()`などで、OnPushと重なります
- `provideZonelessChangeDetection()`で有効にし、Angular 21以降は新規プロジェクトの既定です

次章からは、複数ページを扱うためのAngular Routerに進みます。
