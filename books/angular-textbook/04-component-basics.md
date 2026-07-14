---
title: "TypeScriptとComponentの基本"
---

ここからComponentの世界に入ります。まずComponentを書く土台となるTypeScriptを整理し、続いてComponentの基本と、その現在の標準であるStandaloneを学びます。

:::message
**この章で学ぶこと**

- AngularがTypeScriptを採用している理由
- 型注釈・interface・型エイリアスの基本
- Componentを構成する要素
- `@Component`デコレーターの役割
- NgModuleがどのような仕組みだったか
- NgModuleが抱えていた課題
:::

## Angularで使うTypeScript

Angularは、TypeScriptを前提に設計されたフレームワークです。Componentを書くにも、Serviceを書くにも、TypeScriptの知識が土台になります。この節では、Angularを学ぶうえで欠かせないTypeScriptの要点にしぼって整理します。

TypeScriptを網羅的に解説するわけではありません。あくまで「この先の章を読むために必要な範囲」に絞ります。すでにTypeScriptに慣れている方は、Angularで特によく使う部分を確認するつもりで読み進めてください。

### なぜAngularはTypeScriptなのか

TypeScriptは、JavaScriptに「型」を加えた言語です。JavaScriptとして動くコードに、変数や関数が扱う値の種類を注釈として書き添えます。

型があると、コードを実行する前に多くの誤りを見つけられます。たとえば、文字列を期待している場所に数値を渡してしまった、存在しないプロパティを参照してしまった、といった間違いを、エディタが赤い波線で教えてくれます。アプリケーションが大きくなり、扱うデータや部品が増えるほど、この「実行前に気づける」という性質は大きな助けになります。Angularが中〜大規模の開発で力を発揮するのは、この型の後押しがあるからでもあります。

### 型注釈と基本の型

変数や関数の引数に、コロンに続けて型を書きます。これを型注釈と呼びます。

```ts
const title: string = 'Angular';
const count: number = 0;
const isActive: boolean = true;
const tags: string[] = ['a', 'b', 'c'];
```

もっとも、値から型が明らかな場合は、型注釈を省略できます。TypeScriptが値を見て型を推論してくれるためです。

```ts
const title = 'Angular'; // string と推論される
const count = 0;          // number と推論される
```

Angularのコードでは、この型推論を活かして、注釈を書きすぎないのが一般的です。一方で、関数の引数や戻り値のように、推論だけでは意図が伝わりにくい箇所には、型を明示します。

```ts
function double(value: number): number {
  return value * 2;
}
```

:::message
どんな値でも受け付ける`any`という型もありますが、`any`を使うと型の恩恵が失われます。安易な`any`は避け、適切な型を付けることを心がけてください。
:::

### interfaceと型エイリアス

アプリケーションでは、「ユーザー」や「商品」といった、複数の値をまとめたデータを扱います。そのデータの形を定義するのが、interfaceと型エイリアスです。Angularでは、サーバーから受け取るデータの形を表すのによく使います。

```ts
interface User {
  id: number;
  name: string;
  email: string;
}

const user: User = {
  id: 1,
  name: '山田',
  email: 'yamada@example.com',
};
```

同じことは、型エイリアス（`type`）でも書けます。

```ts
type User = {
  id: number;
  name: string;
  email: string;
};
```

どちらを使っても構いませんが、データの形を表すときはinterfaceを使う流儀が広く見られます。型エイリアスは、複数の型を組み合わせた型など、より柔軟な表現に向いています。

### ユニオン型とリテラル型

値が「複数の型のいずれか」を取りうる場合は、`|`で型を並べたユニオン型で表します。

```ts
let value: string | number;
value = 'text'; // OK
value = 100;    // OK
```

さらにTypeScriptでは、文字列や数値の「特定の値そのもの」を型にできます。これをリテラル型と呼びます。ユニオン型と組み合わせると、「決まった選択肢のいずれか」を表現できます。

```ts
type Status = 'idle' | 'loading' | 'success' | 'error';

let status: Status = 'idle';
status = 'loading'; // OK
status = 'done';    // エラー: 選択肢に含まれない
```

このリテラル型のユニオンは、Angularのアプリケーションで状態を表すのによく使われます。たとえば通信の状態を`'idle' | 'loading' | 'success' | 'error'`で表しておけば、想定外の値が入ることを型が防いでくれます。文字列をそのまま使う場合に比べ、打ち間違いや考慮漏れに気づきやすくなります。

### クラスとアクセス修飾子

Angularでは、Componentもクラスとして書きます。クラスは、データ（プロパティ）と処理（メソッド）をひとまとめにした設計図です。

```ts
class Counter {
  count = 0;

  increment(): void {
    this.count++;
  }
}
```

クラスのプロパティやメソッドには、外部からの見え方を制御するアクセス修飾子を付けられます。

- `public`: どこからでもアクセスできる（省略時のデフォルト）
- `private`: そのクラスの内部からのみアクセスできる
- `protected`: そのクラスと、継承したクラスからアクセスできる

Angularでは、テンプレートから使うプロパティやメソッドを`protected`にし、クラス内部だけで使うものを`private`にする、といった使い分けがよく行われます。また、変更されない値には`readonly`を付けて、あとから書き換えられないようにします。

```ts
class Counter {
  protected readonly count = signal(0);

  protected increment(): void {
    this.count.update((value) => value + 1);
  }
}
```

### ジェネリクス

ジェネリクスは、型を「後から指定できる」ようにする仕組みです。角かっこ（`<>`）の中に型を書いて、扱う値の型を伝えます。

Angularでは、この記法がいたるところに登場します。たとえばSignalは`Signal<number>`のように、中身の型を指定して使います。配列も、`Array<string>`という形はジェネリクスの一種です。

```ts
const count: Signal<number> = signal(0);
const names: Array<string> = ['a', 'b'];
```

いまはすべてを理解する必要はありません。「`<>`の中は、扱う値の型を表している」と読めれば十分です。Signalの詳細は第6部、Observableの`Observable<T>`は第8部で扱います。

サーバー通信でも、`http.get<User[]>(...)`のように、受け取るデータの型をジェネリクスで指定します。型を渡しておくと、返ってきた値がその型として扱われ、後続のコードで補完や型チェックが効くようになります。このように、ジェネリクスはAngularのAPIを使いこなすうえで避けて通れない記法です。最初は`<>`の中身が何を表すのかを読み取れれば十分で、自分で複雑なジェネリクスを書けるようになる必要は、当面ありません。

### デコレーター

Angularのコードには、`@Component`や`@Injectable`のように、`@`から始まる記法が登場します。これをデコレーターと呼びます。デコレーターは、クラスやプロパティに「役割」や「設定（メタデータ）」を与えるための記法です。

```ts
@Component({
  selector: 'app-counter',
  template: `...`,
})
export class Counter {}
```

この例では、`@Component`が「このクラスはComponentである」という役割と、セレクターやテンプレートといった設定を、`Counter`クラスに与えています。Angularは、このデコレーターの情報を読み取って、クラスをComponentとして扱います。デコレーターはAngularのいたるところで使われるので、まずは「クラスに役割や設定を添える記法」として覚えておいてください。

### オプショナルとnull安全

プロパティが「あるかもしれないし、ないかもしれない」場合は、名前のうしろに`?`を付けてオプショナルにします。

```ts
interface User {
  id: number;
  name: string;
  age?: number; // 省略してもよい
}
```

後述するStrictモードでは、`null`や`undefined`かもしれない値を、そのまま使うことができません。値があるときだけプロパティをたどりたい場合は、オプショナルチェーン（`?.`）を使います。途中が`null`や`undefined`なら、そこで評価を止めて`undefined`を返します。

```ts
const length = user.name?.length; // user.name が無ければ undefined
```

値が無いときに既定値で補いたい場合は、null合体演算子（`??`）が便利です。

```ts
const age = user.age ?? 0; // age が null か undefined なら 0
```

これらの記法は、テンプレートの中でも同じように使えます。サーバーから届くデータのように、値の有無が定まらないものを扱うとき、こうした記法が安全なコードを支えます。

### Strictモード

Angular CLIで作られるプロジェクトは、TypeScriptの「Strictモード」が既定で有効になっています。Strictモードでは、型のチェックが厳しくなり、たとえば`null`や`undefined`かもしれない値を、そのまま使うことが許されなくなります。

最初は少し窮屈に感じるかもしれませんが、これは実行時のエラーを未然に防ぐための仕組みです。Strictモードのもとで書かれたコードは、より安全で壊れにくくなります。本書のサンプルコードも、Strictモードで動くことを前提にしています。

:::message
TypeScriptの型は、ビルド時のチェックにのみ使われ、実行時のJavaScriptには残りません。そのため、型を付けていても、サーバーから届く値が本当にその型である保証にはなりません。外部から来るデータは、必要に応じて実行時にも確認する、という意識を持っておくと安全です。
:::

## Componentとは何か

Angularの画面は、Componentという部品を組み合わせて作られます。前節でTypeScriptの土台を整えたので、ここからは実際にComponentの中身を見ていきます。この節では、Componentが何でできているのか、どうやって作るのか、そしてどうつなげて画面を組み立てるのかを学びます。

Componentは、Angularを学ぶうえで最初にして最大の基本です。この先に登場するDirectiveやService、状態管理といった仕組みも、すべてComponentを中心に組み立てられます。ここでの理解が、以降のすべての章の土台になります。

### Componentを構成する要素

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

### @Componentデコレーターの構成

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

### Componentを作る

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

### セレクターとComponentのネスト

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

### セレクターの種類と命名規則

`selector`には、いくつかの指定方法があります。もっとも一般的なのは、これまで見てきた要素型のセレクターです。

- **要素型**（`app-greeting`）: `<app-greeting>`というタグとして使う。Componentの基本
- **属性型**（`[appHighlight]`）: `<p appHighlight>`のように、既存の要素へ属性として付ける。Directiveでよく使う

Componentでは、要素型を使うのが基本です。ここで、セレクターの先頭に`app-`という接頭辞が付いていることに注目してください。これは「自分のアプリのComponentである」ことを示す目印で、標準のHTML要素や、外部ライブラリのComponentと名前がぶつかるのを防ぎます。接頭辞は`ng new`で作ったプロジェクトでは既定で`app`ですが、プロジェクトごとに変更できます。

また、要素型のセレクターは、`user-profile`のようにハイフンでつなぐケバブケースで書きます。これは、ブラウザのカスタム要素の命名規則に合わせたものです。Angular CLIでComponentを生成すれば、この規則に沿ったセレクターが自動で付けられます。

### テンプレートとクラスのつながり

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

### Componentが生まれてから消えるまで

Componentは、画面に表示されるときに生成され、不要になると破棄されます。生成・表示・破棄といった節目には、それぞれ処理を差し込むための仕組みが用意されています。これをライフサイクルと呼びます。

たとえば「Componentが表示された直後にデータを読み込む」「破棄されるときに後始末をする」といった処理を、決められたタイミングで実行できます。ライフサイクルは、入力値の変化とあわせて第21章で詳しく扱います。いまは「Componentには一生（ライフサイクル）がある」ということだけ知っておいてください。

たとえば、`ngOnInit`は生成直後の初期化処理に、`ngOnDestroy`は破棄されるときの後始末に使われます。データの読み込みを`ngOnInit`で行う、といった使い方が代表例です。これらの名前だけ頭の片隅に置いておくと、第21章での理解がスムーズになります。なお、モダンAngularでは、こうした初期化の一部を、SignalやDIの仕組みで置き換えられる場面も増えています。その話題は、該当する章で改めて触れます。

## NgModuleからStandalone Componentへ

前節では、Componentのテンプレートで別の部品を使うために`imports`へ宣言する、という書き方を見ました。この`imports`は、Standalone Componentという仕組みの一部です。この節では、そのStandalone Componentが登場する前にどうしていたのか、つまり従来のNgModuleと比較しながら、なぜいまの形になったのかを掘り下げます。

現在の新規開発では、NgModuleを書く場面はほとんどありません。しかし、少し前のプロジェクトや、Web上の記事にはNgModuleが数多く登場します。両者の関係を理解しておくと、古いコードを読むときにも、移行を進めるときにも役立ちます。

### NgModuleとは何だったか

かつてのAngularでは、Component・Directive・Pipeを、単体で使うことができませんでした。これらは必ず、NgModuleという入れ物に登録する必要がありました。NgModuleは、`@NgModule`デコレーターを付けたクラスで、次のような設定を持ちます。

```ts:旧来の書き方（NgModule）
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Greeting } from './greeting';

@NgModule({
  declarations: [Greeting],
  imports: [CommonModule],
  exports: [Greeting],
})
export class GreetingModule {}
```

それぞれの設定には、次の役割があります。

- **declarations**: このモジュールに属するComponent・Directive・Pipeを登録する
- **imports**: このモジュールが必要とする、別のモジュールを取り込む
- **exports**: 外部のモジュールに公開する部品を指定する
- **providers**: DIで注入するServiceを登録する

NgModuleは、関連する部品をひとまとめにして、どれを外部に公開し、どれを内部にとどめるのかを管理する仕組みでした。中規模以上のアプリケーションを整理するうえでは、たしかに役立ちました。

### NgModuleが抱えていた課題

一方で、NgModuleには扱いにくさもありました。

まず、Componentを1つ作るたびに、それをどこかのNgModuleの`declarations`に登録しなければなりませんでした。小さな部品を1つ足すだけでも、Component本体とNgModuleの2か所を触ることになります。初学者にとっては、「なぜこの登録がいるのか」が分かりにくく、理解の壁になっていました。

次に、ある部品を使いたいとき、それがどのNgModuleに属していて、どのモジュールを`imports`すればよいのかを追う必要がありました。この依存関係の見通しの悪さも、開発の負担になっていました。

つまりNgModuleは、部品を束ねる便利さと引き換えに、記述の手間と理解の難しさを生んでいたのです。

### Standalone Componentによる解決

この課題に応えて登場したのが、Standalone Componentです。Standalone Componentは、NgModuleという入れ物を必要とせず、単体で成立します。自身が必要とする部品を、`@Component`の`imports`に直接宣言するのが特徴です。

```ts:src/app/greeting.ts
import { Component } from '@angular/core';
import { UserIcon } from './user-icon';

@Component({
  selector: 'app-greeting',
  imports: [UserIcon],
  template: `
    <app-user-icon />
    <p>こんにちは</p>
  `,
})
export class Greeting {}
```

必要な依存が、Componentの中に閉じて宣言されているため、そのComponentが何に依存しているのかがひと目でわかります。NgModuleという中間層がなくなり、部品とその依存の関係がまっすぐに見えるようになりました。

Standalone Componentは、Angular 14（2022年）でプレビューとして登場し、Angular 17（2023年11月）で新規プロジェクトの標準的な生成形式になりました。さらにAngular 19（2024年11月）では、`standalone: true`という指定すら不要になり、Standaloneであることが暗黙のデフォルトになっています。そのため、現在のコードでは`standalone: true`という記述を書きません。

:::message
少し前のプロジェクトでは、`@Component`に`standalone: true`と明記されていることがあります。これはStandaloneであることを示す当時の書き方で、Angular 19以降では省略できます。逆に`standalone: false`と書かれている場合は、NgModuleに登録して使う従来のComponentを表します。
:::

Standaloneには、見通しのよさ以外の利点もあります。各Componentが自分の依存を把握しているため、必要になったときにだけ読み込む「遅延読み込み」がしやすくなります。結果として、最初に読み込むコードの量を減らし、アプリの初期表示を速くしやすくなります。遅延読み込みは、第7部のルーティングで詳しく扱います。

### 新旧のコードを比べる

同じ「Greetingを画面に表示する」構成を、NgModule時代とStandalone時代で比べてみます。まずNgModule時代は、Componentのほかに、それを束ねるモジュールが必要でした。

```ts:旧来の書き方（NgModule時代）
// greeting.ts と greeting.module.ts の2ファイルが必要
@NgModule({
  declarations: [Greeting],
  exports: [Greeting],
})
export class GreetingModule {}
```

Standalone時代では、モジュールが不要になり、Component1つで完結します。

```ts:src/app/greeting.ts（現在の書き方）
@Component({
  selector: 'app-greeting',
  imports: [/* 必要な部品 */],
  templateUrl: './greeting.html',
})
export class Greeting {}
```

ファイル数が減り、依存関係がComponentの中に集約されているのがわかります。

### モジュールから関数へ

NgModuleは、部品を束ねるだけでなく、アプリ全体の設定を登録する役割も担っていました。この役割も、モダンAngularでは形を変えています。

たとえば、ルーティングを使うには、かつては`RouterModule.forRoot(routes)`をNgModuleの`imports`に登録していました。現在は、第4章で見たように、`app.config.ts`で`provideRouter(routes)`という関数を使います。同じように、HTTP通信も`HttpClientModule`のimportから`provideHttpClient()`へと変わりました。「モジュールをimportする」書き方から、「`provide〜`という関数を並べる」書き方へ移ったのです。

```ts:旧来の書き方（NgModuleでの設定登録）
@NgModule({
  imports: [RouterModule.forRoot(routes), HttpClientModule],
})
export class AppModule {}
```

```ts:src/app/app.config.ts（現在の書き方）
export const appConfig: ApplicationConfig = {
  providers: [provideRouter(routes), provideHttpClient()],
};
```

また、`*ngIf`や`*ngFor`を使うために必要だった`CommonModule`のimportも、現在は不要です。条件分岐や繰り返しは、`@if`や`@for`という組み込みの構文に置き換わり、importなしで使えるようになりました。これらの新しい構文は、第3部で詳しく扱います。

かつては、よく使う部品をまとめた`SharedModule`を作り、各所でimportする手法も一般的でした。Standaloneでは、必要なComponentやDirectiveを、使う側が直接`imports`に書きます。共有のための中間モジュールを設けなくても、部品を個別に取り込めます。

### 既存プロジェクトを移行する

NgModuleで書かれた既存のプロジェクトは、公式の移行ツール（スキマティクス）を使って、Standaloneへ自動で変換できます。次のコマンドで実行します。

```bash
ng generate @angular/core:standalone
```

この移行は、大きく3つの段階で進みます。

1. すべてのComponent・Directive・PipeをStandaloneにし、必要な依存を各Componentの`imports`へ移す
2. 不要になったNgModule（部品を再公開するだけのもの）を削除する
3. 起動処理を`NgModule`ベースから`bootstrapApplication`へ置き換える

自動で判断できない部分には印（TODOコメント）が残されるため、そこは手作業で確認します。すべてを一度に変換しようとせず、段階的に進められるのも、この移行ツールの特徴です。NgModuleとStandaloneは共存できるため、大きなプロジェクトでも、機能ごとに少しずつ移行を進められます。一度にすべてを書き換える必要はありません。

:::message
移行ツールを実行する前に、変更をコミットしておくと安心です。自動変換の結果を差分で確認しながら、段階的に進めましょう。うまく変換できなかった箇所は、残されたTODOコメントを手がかりに、手作業で対応します。
:::

## まとめ

- TypeScriptはJavaScriptに型を加えた言語で、実行前に多くの誤りを見つけられます
- データの形はinterfaceや型エイリアスで定義し、サーバーから受け取る値の型などに使います
- Componentはクラスで書き、アクセス修飾子や`readonly`で見え方や書き換えを制御します
- Componentは、テンプレート・クラス・スタイルの3要素と、それらを束ねる`@Component`から成ります
- `selector`はComponentを呼び出すためのタグ名で、`imports`はテンプレートで使う部品の宣言です
- Angular CLIの`ng generate component`で、ファイル一式を生成できます
- NgModuleは、Component・Directive・Pipeを束ねて管理する従来の仕組みでした
- 便利な一方で、登録の手間や依存関係の見通しの悪さという課題を抱えていました
- Standalone Componentは、必要な依存を`imports`に直接宣言し、単体で成立します

次章では、Componentを組み合わせ、洗練させるための構成技法と設計を扱います。
