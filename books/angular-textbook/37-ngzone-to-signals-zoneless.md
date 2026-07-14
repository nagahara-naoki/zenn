---
title: "NgZoneからSignals・Zonelessへ"
---

第6部の締めくくりとして、変更検知の大きな転換点であるZonelessを学びます。第28章で、Zone.jsが変更検知を起動していたこと、そしてその課題を見ました。第29章では、Signalが変更検知の起点になれることを見ました。この2つを結ぶと、ひとつの結論が導かれます。「Signalがあれば、Zone.jsは要らないのではないか」。この発想を実現したのが、Zonelessです。

Zonelessは、Zone.jsを取り除いてアプリケーションを動かす方式です。実験的な導入を経て、Angular 20.2で安定版となり、Angular 21（2025年）では新規プロジェクトの既定になりました。本書が基準とするv22世代では、Zonelessが標準的な選択肢であり、これから学ぶ書き方も、すべてZonelessを前提として問題なく動きます。この章では、Zone.js方式とZoneless方式を比較し、何が変わり、開発者は何を意識すればよいのかを整理します。

:::message
**この章で学ぶこと**

- ZonelessとZone.js方式の違い
- Zonelessで変更検知が走る条件
- Zonelessを有効にする方法
- Zonelessがもたらす利点と、移行の指針
:::

## Zonelessとは何か

Zonelessとは、その名のとおり「ゾーン（Zone.js）がない」状態でAngularを動かすことです。第28章で見たように、従来はZone.jsが非同期の出来事を監視し、それを合図に変更検知を起動していました。Zonelessでは、この監視役がいません。代わりに、「状態が変わった」という明示的な通知を受けて、変更検知が走ります。

考え方の違いを、ひとことで表せます。Zone.js方式は「何か非同期が起きた。念のため確認しよう」という方式でした。Zoneless方式は「ここが変わった、と知らされた。そこを更新しよう」という方式です。前者は起こりうる変化を広く拾い、後者は実際の変化を正確に受け取ります。この違いが、効率の差を生みます。

## Zonelessで変更検知が走る条件

Zone.jsがいないと、Angularはどうやって「状態が変わった」ことを知るのでしょうか。Zonelessでは、次のような明示的な通知が、変更検知の起点になります。

- **Signalの更新**: テンプレートで読まれているSignalが変わると、Angularに通知されます
- **`AsyncPipe`の発火**: `async`パイプが新しい値を受け取ると、更新が要求されます
- **イベントリスナー**: テンプレートやホストのイベントハンドラーが実行されると、検知が走ります
- **`markForCheck()`の呼び出し**: `ChangeDetectorRef`で明示的に更新を要求した場合
- **`setInput()`**: 動的に生成したComponentへ入力を設定した場合

見比べてわかるとおり、これらは第27章で挙げたOnPushの更新条件と、ほぼ重なります。実は、Zonelessは「アプリ全体がOnPushで動く」ようなものだと考えられます。だからこそ、前章までで学んだOnPushとSignalの組み合わせが、そのままZonelessの土台になるのです。

とりわけ重要なのがSignalです。Signalで状態を持っていれば、その変化は自動でAngularに通知されます。開発者が`markForCheck()`を呼ぶ必要はありません。Zonelessで快適に開発する鍵は、状態をSignalで管理することにあります。

## Zonelessを有効にする

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

## 新旧を比べる

Zone.js方式とZoneless方式の違いを、表に整理します。

| 観点 | Zone.js方式（旧） | Zoneless方式（新） |
|---|---|---|
| 変更検知の起点 | 非同期の出来事をZone.jsが検知 | Signalなどの明示的な通知 |
| 検知の範囲 | 広く（起きうる変化を拾う） | 狭く（実際の変化を受け取る） |
| バンドルサイズ | Zone.jsの分だけ大きい | Zone.jsがなく小さい |
| デバッグ | スタックトレースが複雑 | 素直なスタックトレース |
| 状態管理の前提 | 任意（プロパティでも動く） | Signalが基本 |

Zonelessは、Zone.jsの4つの課題――過剰な変更検知、バンドルサイズ、デバッグの難しさ、エコシステムとの相性――を、まとめて解消します。Zone.jsという監視の層がなくなることで、アプリは軽く、速く、追いやすくなります。

## Zonelessへの移行の指針

これから新しくアプリを作るなら、Zonelessが既定であり、特別な準備は要りません。状態をSignalで持ち、非同期の表示には`async`パイプを使う、という本書で学んできた書き方が、そのままZonelessに適合します。

一方、Zone.jsに依存した既存プロジェクトを移行する場合は、いくつか確認すべき点があります。たとえば、`setTimeout`のコールバックの中でプロパティを書き換え、Zone.jsによる自動検知に頼って画面を更新していた箇所は、Zonelessでは更新されないことがあります。こうした箇所は、状態をSignalに置き換えるのが、もっとも素直な対処です。第28章で触れた`runOutsideAngular`のような、Zone.jsを前提とした最適化も、Zonelessでは不要になり、書き換えの対象になります。

移行は一度に済ませる必要はありません。まずOnPushとSignalへ寄せてアプリを整え、Zone.jsへの依存を減らしてから、Zonelessに切り替える、という段階的な進め方が現実的です。本書がSignalファーストで書いてきたのは、この移行のしやすさも見据えてのことでした。

## 新旧のコードを比べる

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

## ハイブリッドな移行

大規模な既存アプリを、一度に完全なZonelessへ移すのは容易ではありません。そのため、段階的な移行の道も用意されています。Zone.jsを残したまま、Signalによる効率的な変更検知の恩恵を先に受け、準備が整った部分からZone.jsへの依存を外していく、という進め方です。

現実的な手順としては、次のようになります。まず、状態をSignalへ、非同期の表示を`async`パイプへ置き換えます。次に、`setTimeout`のコールバック内での直接的なプロパティ書き換えなど、Zone.jsの自動検知に暗黙的に頼った箇所を洗い出し、Signalベースに直します。最後に、Zone.jsを取り除いてZonelessに切り替え、動作を確認します。焦らず、一部分ずつ検証しながら進めるのが安全です。

:::message alert
既存アプリからZone.jsを取り除く前に、Zone.jsの自動検知に依存した箇所が残っていないかを十分に確認してください。見落とすと、特定の操作で画面が更新されない不具合につながります。状態をSignalへ移す作業を先に済ませてから、Zone.jsの除去に着手するのが安全です。
:::

:::message
Zonelessは、Signalで状態を管理していれば、多くの場合そのまま動きます。逆に、Zone.jsの自動検知に暗黙的に頼ったコードは、移行時に見直しが必要です。日ごろからSignalファーストで書いておくことが、Zoneless時代への最良の備えになります。
:::

## まとめ

- Zonelessは、Zone.jsを取り除き、明示的な通知で変更検知を走らせる方式です
- 起点はSignalの更新・`async`パイプ・イベント・`markForCheck()`などで、OnPushと重なります
- `provideZonelessChangeDetection()`で有効にし、Angular 21以降は新規プロジェクトの既定です
- Zonelessは、過剰な検知・バンドルサイズ・デバッグ・相性の課題を解消します
- Zone.jsに依存した既存コードは、状態をSignalへ移すことでZonelessに適合します
- **新規開発ではZonelessが標準です。状態をSignalで持つことが、その前提であり最良の備えです**

以上で第6部は終わりです。次の第7部では、複数のページを扱うアプリケーションに欠かせない、Angular Routerによる画面遷移を学びます。
