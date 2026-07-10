---
title: "constructorの内部構造：ECMAScript仕様とV8実装を分けて理解する"
emoji: "🤖"
type: "tech"
topics: ["javascript", "ecmascript", "v8", "constructor"]
published: true
---

JavaScriptのconstructorを深く理解するときは、次の2層を分ける必要があります。

1. **ECMAScript仕様**：JavaScriptとして観測できる動作を定義する
2. **エンジン実装**：V8などが、その動作をどう高速に実現するか

仕様に `[[Construct]]` や `[[Prototype]]` は登場しますが、V8のMap（Hidden Class）やメモリ配置は実装上の仕組みです。両者を同じものとして説明すると、理解を誤ります。

## 対象とするコード

```js
class User {
  role = "member";

  constructor(name) {
    this.name = name;
  }

  greet() {
    return `Hello, ${this.name}`;
  }
}

const user = new User("Taro");
```

この記事では、`new User("Taro")` から `user` が得られるまでを追います。

## class定義を評価した時点

`new` より前に、class定義の評価によってconstructor function objectが作られます。

概念上、`User` は次のような関係を持ちます。

```text
User  ── prototype property ──> User.prototype
                                  ├─ constructor ──> User
                                  └─ greet()
```

`greet` はインスタンスへコピーされるのではなく、`User.prototype` のメソッドとして定義されます。

class fieldの `role` は、prototype上の共有値ではありません。インスタンス初期化時に、各インスタンスへ定義されます。

## new式は[[Construct]]を呼ぶ

`new User("Taro")` を評価すると、仕様上は大きく次の処理へ進みます。

1. `User` を評価する
2. `User` がconstructorか確認する
3. 引数 `"Taro"` を評価する
4. `User.[[Construct]]` を呼ぶ

`[[Construct]]` は、JavaScriptから直接呼び出すメソッドではなく、仕様上の内部メソッドです。

通常関数のすべてが `[[Construct]]` を持つわけではありません。アロー関数やメソッド定義で作られた関数は、通常constructableではありません。

## 基底クラスのインスタンス生成

`extends` していないclassはbase constructorです。

仕様の詳細を学習用にまとめると、次の流れになります。

```text
1. User.prototypeを基に新しいオブジェクトを作る
2. インスタンス要素を初期化する
   - public fields
   - private fields / methods
3. constructor本体を、新しいオブジェクトをthisとして評価する
4. constructorが別オブジェクトを返さなければthisを返す
```

対象コードでは、インスタンスfieldの初期化により `role` が定義され、その後constructor本体の `this.name = name` が評価されます。

```js
console.log(Object.hasOwn(user, "role")); // true
console.log(Object.hasOwn(user, "name")); // true
console.log(Object.hasOwn(user, "greet")); // false
```

## プロパティ代入とプロパティ定義は同じではない

class fieldは、仕様上 `CreateDataPropertyOrThrow` 相当の処理でインスタンスへ定義されます。

一方、constructor内の通常の代入 `this.name = name` は `[[Set]]` セマンティクスを使います。

この差は、基底クラス側にsetterがある場合などに観測できます。

```js
class Base {
  set value(nextValue) {
    console.log("setter", nextValue);
  }
}

class WithField extends Base {
  value = 1; // setterを呼ばず、自身のdata propertyを定義
}

class WithAssignment extends Base {
  constructor() {
    super();
    this.value = 1; // 継承したsetterが呼ばれる
  }
}
```

「fieldはconstructor先頭への単純な代入に変換される」とだけ覚えると、この違いを説明できません。

## 派生クラスではthisの作られ方が違う

`extends` したclassはderived constructorです。

```js
class Admin extends User {
  level = 1;

  constructor(name) {
    super(name);
    this.auditEnabled = true;
  }
}
```

derived constructorへ入った時点では、`this` はまだ初期化されていません。

`super(name)` が親constructorをconstructし、その結果を現在の `this` として束縛します。その後、派生クラス側のインスタンスfield（ここでは `level`）が初期化され、残りのconstructor本体が進みます。

```text
Admin constructorへ入る
  ↓
super(name)
  ├─ User側のインスタンス要素を初期化
  ├─ User constructorを実行
  └─ 結果をAdmin側のthisへ束縛
  ↓
Admin側のインスタンス要素を初期化
  ↓
this.auditEnabled = true
```

このため、`super()` より前に `this` を使うとReferenceErrorになります。

## constructorの戻り値

base constructorがオブジェクトを明示的に返すと、それが構築結果になります。プリミティブ値は通常無視されます。

derived constructorは、オブジェクトまたは `undefined` 以外を返すとTypeErrorになるなど、規則が異なります。

実務ではconstructorから別オブジェクトを返さず、生成結果を切り替える場合はfactoryを使う方が読みやすくなります。

## user.constructorはどこから来るか

```js
console.log(user.constructor === User); // true
```

`user` 自身に `constructor` があるわけではありません。

`user.[[Prototype]]` が `User.prototype` を参照し、そこにある `constructor` プロパティが見つかります。

```js
console.log(Object.hasOwn(user, "constructor")); // false
console.log(User.prototype.constructor === User); // true
```

`constructor` プロパティは書き換え可能なので、信頼できない値の型判定へ無条件に使うものではありません。

## ここからはV8の実装

ECMAScript仕様は、オブジェクトを特定のC構造体やハッシュマップで実装するよう要求していません。

V8では、同じ形のオブジェクトへ高速にアクセスするため、Map（Hidden Classとも呼ばれる）やDescriptorArrayなどの仕組みを使います。

```js
function createUser(name, age) {
  return { name, age };
}

const a = createUser("Taro", 20);
const b = createUser("Hanako", 25);
```

`a` と `b` は同じ順序で同じプロパティを持つため、V8が形に関する情報を共有し、最適化へ利用できる可能性があります。

ただし、次のような断定はできません。

- すべてのオブジェクトが常に同じ内部表現を使う
- プロパティが必ず特定のメモリ位置へ保存される
- Hidden Classの遷移がコード上の代入と常に1対1で起きる

最適化はV8のバージョン、実行履歴、プロパティ種別などで変わります。業務コードでは、推測でマイクロ最適化する前に計測します。

## プロパティ順を揃えるべきか

同種オブジェクトを一貫した形で生成することは、可読性と型の予測可能性に役立ち、エンジン最適化にも有利になり得ます。

ただし、constructorの役割はHidden Classを手動管理することではありません。

優先順位は次の通りです。

1. インスタンスの前提条件を明確にする
2. すべてのインスタンスを一貫した状態にする
3. 実測で問題がある場合だけ、対象エンジンの挙動を調べる

## まとめ

`new User()` の理解には、仕様と実装を分けることが重要です。

- `new` はconstructorの `[[Construct]]` を呼ぶ
- base constructorでは新しいオブジェクトとインスタンス要素が準備される
- class fieldの定義とconstructor内の代入はセマンティクスが異なる
- derived constructorの `this` は `super()` によって初期化される
- prototypeメソッドはインスタンスへコピーされない
- V8のMapは仕様ではなく、観測可能な動作を高速に実現するための実装

## 参考資料

- [ECMAScript Language Specification: Function [[Construct]]](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ecmascript-function-objects-construct-argumentslist-newtarget)
- [ECMAScript Language Specification: InitializeInstanceElements](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-initializeinstanceelements)
- [MDN: Public class fields](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Classes/Public_class_fields)
- [V8: Fast properties in V8](https://v8.dev/blog/fast-properties)
