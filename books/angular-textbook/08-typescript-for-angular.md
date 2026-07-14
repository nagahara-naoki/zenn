---
title: "第5章 Angularで使うTypeScript"
---

Angularは、TypeScriptを前提に設計されたフレームワークです。Componentを書くにも、Serviceを書くにも、TypeScriptの知識が土台になります。この章では、Angularを学ぶうえで欠かせないTypeScriptの要点にしぼって整理します。

TypeScriptを網羅的に解説するわけではありません。あくまで「この先の章を読むために必要な範囲」に絞ります。すでにTypeScriptに慣れている方は、Angularで特によく使う部分を確認するつもりで読み進めてください。

:::message
**この章で学ぶこと**

- AngularがTypeScriptを採用している理由
- 型注釈・interface・型エイリアスの基本
- クラスとアクセス修飾子
- ジェネリクスとデコレーターの考え方
:::

## なぜAngularはTypeScriptなのか

TypeScriptは、JavaScriptに「型」を加えた言語です。JavaScriptとして動くコードに、変数や関数が扱う値の種類を注釈として書き添えます。

型があると、コードを実行する前に多くの誤りを見つけられます。たとえば、文字列を期待している場所に数値を渡してしまった、存在しないプロパティを参照してしまった、といった間違いを、エディタが赤い波線で教えてくれます。アプリケーションが大きくなり、扱うデータや部品が増えるほど、この「実行前に気づける」という性質は大きな助けになります。Angularが中〜大規模の開発で力を発揮するのは、この型の後押しがあるからでもあります。

## 型注釈と基本の型

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

## interfaceと型エイリアス

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

## ユニオン型とリテラル型

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

## クラスとアクセス修飾子

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

## ジェネリクス

ジェネリクスは、型を「後から指定できる」ようにする仕組みです。角かっこ（`<>`）の中に型を書いて、扱う値の型を伝えます。

Angularでは、この記法がいたるところに登場します。たとえばSignalは`Signal<number>`のように、中身の型を指定して使います。配列も、`Array<string>`という形はジェネリクスの一種です。

```ts
const count: Signal<number> = signal(0);
const names: Array<string> = ['a', 'b'];
```

いまはすべてを理解する必要はありません。「`<>`の中は、扱う値の型を表している」と読めれば十分です。Signalの詳細は第6部、Observableの`Observable<T>`は第8部で扱います。

サーバー通信でも、`http.get<User[]>(...)`のように、受け取るデータの型をジェネリクスで指定します。型を渡しておくと、返ってきた値がその型として扱われ、後続のコードで補完や型チェックが効くようになります。このように、ジェネリクスはAngularのAPIを使いこなすうえで避けて通れない記法です。最初は`<>`の中身が何を表すのかを読み取れれば十分で、自分で複雑なジェネリクスを書けるようになる必要は、当面ありません。

## デコレーター

Angularのコードには、`@Component`や`@Injectable`のように、`@`から始まる記法が登場します。これをデコレーターと呼びます。デコレーターは、クラスやプロパティに「役割」や「設定（メタデータ）」を与えるための記法です。

```ts
@Component({
  selector: 'app-counter',
  template: `...`,
})
export class Counter {}
```

この例では、`@Component`が「このクラスはComponentである」という役割と、セレクターやテンプレートといった設定を、`Counter`クラスに与えています。Angularは、このデコレーターの情報を読み取って、クラスをComponentとして扱います。デコレーターはAngularのいたるところで使われるので、まずは「クラスに役割や設定を添える記法」として覚えておいてください。

## オプショナルとnull安全

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

## Strictモード

Angular CLIで作られるプロジェクトは、TypeScriptの「Strictモード」が既定で有効になっています。Strictモードでは、型のチェックが厳しくなり、たとえば`null`や`undefined`かもしれない値を、そのまま使うことが許されなくなります。

最初は少し窮屈に感じるかもしれませんが、これは実行時のエラーを未然に防ぐための仕組みです。Strictモードのもとで書かれたコードは、より安全で壊れにくくなります。本書のサンプルコードも、Strictモードで動くことを前提にしています。

:::message
TypeScriptの型は、ビルド時のチェックにのみ使われ、実行時のJavaScriptには残りません。そのため、型を付けていても、サーバーから届く値が本当にその型である保証にはなりません。外部から来るデータは、必要に応じて実行時にも確認する、という意識を持っておくと安全です。
:::

## まとめ

- TypeScriptはJavaScriptに型を加えた言語で、実行前に多くの誤りを見つけられます
- データの形はinterfaceや型エイリアスで定義し、サーバーから受け取る値の型などに使います
- Componentはクラスで書き、アクセス修飾子や`readonly`で見え方や書き換えを制御します
- `Signal<number>`のようなジェネリクスや、`@Component`のようなデコレーターは、Angularのいたるところで登場します
- Angularのプロジェクトは既定でStrictモードが有効で、より安全なコードになります

次章では、いよいよAngularの主役であるComponentそのものを取り上げ、その構成要素と作り方を学びます。
