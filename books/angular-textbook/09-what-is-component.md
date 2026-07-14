---
title: "第6章 Componentとは何か"
---

Angularの画面は、Componentという部品を組み合わせて作られます。前章でTypeScriptの土台を整えたので、ここからは実際にComponentの中身を見ていきます。この章では、Componentが何でできているのか、どうやって作るのか、そしてどうつなげて画面を組み立てるのかを学びます。

Componentは、Angularを学ぶうえで最初にして最大の基本です。この先に登場するDirectiveやService、状態管理といった仕組みも、すべてComponentを中心に組み立てられます。ここでの理解が、以降のすべての章の土台になります。

:::message
**この章で学ぶこと**

- Componentを構成する要素
- `@Component`デコレーターの役割
- Componentの作り方とファイル構成
- セレクターを使ったComponentのネスト
:::

## Componentを構成する要素

ひとつのComponentは、大きく3つの要素からできています。

- **テンプレート**: 画面の見た目を定義するHTML
- **クラス**: 振る舞いやデータを定義するTypeScriptのクラス
- **スタイル**: 見た目を整えるCSS

そして、これら3つを「これはComponentである」とAngularに伝え、結びつけているのが`@Component`デコレーターです。次のコードは、もっとも基本的なComponentの形です。

```ts:src/app/greeting.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-greeting',
  template: `<p>こんにちは、Angular</p>`,
})
export class Greeting {}
```

`@Component`の中に書かれた設定（メタデータ）が、このクラスをComponentとして成り立たせています。順に見ていきましょう。

## @Componentデコレーターの構成

`@Component`には、Componentの性質を決めるさまざまな設定を書きます。よく使うものは次のとおりです。

- **selector**: このComponentを表すタグ名。ほかのテンプレートから呼び出すときに使います。
- **template / templateUrl**: テンプレートを指定します。短いものは`template`に直接書き、長いものは`templateUrl`で別ファイルを指定します。
- **styleUrl / styles**: スタイルを指定します。こちらも別ファイルか直接記述かを選べます。
- **imports**: このテンプレートで使う、ほかのComponentやDirective、Pipeを宣言します。

テンプレートやスタイルを別ファイルに分ける場合は、次のように書きます。実務では、こちらの形が一般的です。

```ts:src/app/greeting.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-greeting',
  templateUrl: './greeting.html',
  styleUrl: './greeting.css',
})
export class Greeting {}
```

```html:src/app/greeting.html
<p>こんにちは、Angular</p>
```

## Componentを作る

Componentは、第3章で学んだAngular CLIで生成できます。次のコマンドを実行します。

```bash
ng generate component greeting
```

すると、次の4つのファイルが生成されます。

```text
src/app/greeting/
├── greeting.ts        … クラス（振る舞い）
├── greeting.html      … テンプレート（見た目）
├── greeting.css       … スタイル
└── greeting.spec.ts   … テスト
```

ここで、ファイル名が`greeting.ts`、クラス名が`Greeting`となっている点に注目してください。第4章でも触れたとおり、モダンAngularのスタイルガイドでは、ファイル名に`.component`のような接尾辞を付けません。古いプロジェクトで見かける`greeting.component.ts`・`GreetingComponent`と、指しているものは同じです。

テンプレートとスタイルは、`template`・`styles`に直接書くインライン形式と、`templateUrl`・`styleUrl`で別ファイルに分ける形式のどちらでも書けます。数行の短いものは直接書くほうが見通しがよく、長くなるものは別ファイルに分けるほうが読みやすくなります。Angular CLIで生成すると、既定では別ファイルに分かれた形になります。本書では、要点を示すために、短い例ではインライン形式を使うことがあります。

## セレクターとComponentのネスト

Componentの真価は、組み合わせられることにあります。あるComponentのテンプレートの中で、別のComponentをタグとして呼び出せます。これをネスト（入れ子）と呼びます。

呼び出すときの鍵になるのが、`selector`です。先ほどの`Greeting`は`selector: 'app-greeting'`だったので、ほかのComponentのテンプレートに`<app-greeting>`と書くと、その位置に`Greeting`が描画されます。

```ts:src/app/app.ts
import { Component } from '@angular/core';
import { Greeting } from './greeting';

@Component({
  selector: 'app-root',
  imports: [Greeting],
  template: `
    <h1>マイアプリ</h1>
    <app-greeting></app-greeting>
  `,
})
export class App {}
```

ここで重要なのが`imports: [Greeting]`です。Standalone Componentでは、テンプレートで使う別のComponentを、この`imports`に宣言する必要があります。宣言を忘れると、`<app-greeting>`はただの不明なタグとして扱われ、画面には何も表示されません。

:::message
テンプレートに置いたComponentが表示されないときは、まず`imports`への宣言を確認してください。これは初学者がつまずきやすい典型的なポイントです。
:::

なお、子要素を持たないComponentは、`<app-greeting></app-greeting>`と書く代わりに、`<app-greeting />`と自己完結型のタグで書くこともできます。本書でも、内容を差し込まないComponentは、この短い形で書くことがあります。

## セレクターの種類と命名規則

`selector`には、いくつかの指定方法があります。もっとも一般的なのは、これまで見てきた要素型のセレクターです。

- **要素型**（`app-greeting`）: `<app-greeting>`というタグとして使う。Componentの基本
- **属性型**（`[appHighlight]`）: `<p appHighlight>`のように、既存の要素へ属性として付ける。Directiveでよく使う

Componentでは、要素型を使うのが基本です。ここで、セレクターの先頭に`app-`という接頭辞が付いていることに注目してください。これは「自分のアプリのComponentである」ことを示す目印で、標準のHTML要素や、外部ライブラリのComponentと名前がぶつかるのを防ぎます。接頭辞は`ng new`で作ったプロジェクトでは既定で`app`ですが、プロジェクトごとに変更できます。

また、要素型のセレクターは、`user-profile`のようにハイフンでつなぐケバブケースで書きます。これは、ブラウザのカスタム要素の命名規則に合わせたものです。Angular CLIでComponentを生成すれば、この規則に沿ったセレクターが自動で付けられます。

## テンプレートとクラスのつながり

テンプレートとクラスは、たがいに連携します。クラスが持つデータをテンプレートに表示したり、テンプレートでの操作をクラスのメソッドで受け取ったりできます。次のコードは、クラスの`name`をテンプレートに表示する例です。

```ts:src/app/greeting.ts
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-greeting',
  template: `<p>こんにちは、{{ name() }}さん</p>`,
})
export class Greeting {
  protected readonly name = signal('山田');
}
```

`{{ name() }}`は補間（interpolation）と呼ばれる記法で、クラスの値をテンプレートに埋め込みます。ここでは`name`がSignalなので、`name()`と呼び出して値を取り出しています。このようなテンプレートとクラスのやり取り、いわゆるデータバインディングは、次の第3部で本格的に扱います。ここでは「クラスとテンプレートは連携する」という点だけ押さえておけば十分です。

## Componentが生まれてから消えるまで

Componentは、画面に表示されるときに生成され、不要になると破棄されます。生成・表示・破棄といった節目には、それぞれ処理を差し込むための仕組みが用意されています。これをライフサイクルと呼びます。

たとえば「Componentが表示された直後にデータを読み込む」「破棄されるときに後始末をする」といった処理を、決められたタイミングで実行できます。ライフサイクルは、入力値の変化とあわせて第21章で詳しく扱います。いまは「Componentには一生（ライフサイクル）がある」ということだけ知っておいてください。

たとえば、`ngOnInit`は生成直後の初期化処理に、`ngOnDestroy`は破棄されるときの後始末に使われます。データの読み込みを`ngOnInit`で行う、といった使い方が代表例です。これらの名前だけ頭の片隅に置いておくと、第21章での理解がスムーズになります。なお、モダンAngularでは、こうした初期化の一部を、SignalやDIの仕組みで置き換えられる場面も増えています。その話題は、該当する章で改めて触れます。

## まとめ

- Componentは、テンプレート・クラス・スタイルの3要素と、それらを束ねる`@Component`から成ります
- `selector`はComponentを呼び出すためのタグ名で、`imports`はテンプレートで使う部品の宣言です
- Angular CLIの`ng generate component`で、ファイル一式を生成できます
- 別のComponentをテンプレートで使うには、`imports`への宣言が必要です
- テンプレートとクラスは連携し、その仕組み（データバインディング）は第3部で扱います

次章では、この`imports`による依存の宣言がどこから来たのか、従来のNgModuleと現在のStandalone Componentを比較しながら掘り下げます。
