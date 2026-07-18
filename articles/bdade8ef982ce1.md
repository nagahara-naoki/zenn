---
title: "constructorの内部構造：ECMAScript仕様とV8実装を分けて読む"
emoji: "🤖"
type: "tech"
topics: ["javascript", "ecmascript", "v8", "constructor"]
published: true
---

constructorの内部を読むときは、ECMAScript仕様とエンジン実装を分けます。仕様は観測可能な結果と順序を定め、V8はChromeやNode.jsでそれを実現する一つの実装です。

- `[[Construct]]`、`NewTarget`、`[[Prototype]]` は仕様上の概念
- Map（Hidden Class）、DescriptorArray、inline cache、Ignition、TurboFanはV8固有の用語

したがって、「constructorがHidden Classを作る」「`[[Prototype]]` は必ずこのメモリ位置にある」とは言えません。この記事は仕様が保証する動作を先に追い、その後にV8の最適化を重ねます。

## 観測対象となるclass

次のコードを一貫した題材にします。

```js
class User {
  role = "member";
  #token = crypto.randomUUID();

  constructor(name) {
    this.name = name.trim();
  }

  greet() {
    return `Hello, ${this.name}`;
  }

  hasToken() {
    return this.#token.length > 0;
  }
}

const user = new User(" Taro ");
```

実行後に標準APIから観測できる事実は次のとおりです。

```js
console.log(Object.keys(user)); // ["role", "name"]
console.log(Object.getPrototypeOf(user) === User.prototype); // true
console.log(Object.hasOwn(user, "greet")); // false
console.log(Object.hasOwn(User.prototype, "greet")); // true
console.log(user.hasToken()); // true
```

private fieldの `#token` は通常のproperty keyとして扱われず、`Object.keys`、`Reflect.ownKeys`、property descriptorから列挙できません。ここから先は「なぜこの観測結果になるか」を仕様層で説明します。

## 仕様層：class定義の評価

class定義を評価すると、constructor function objectとprototype objectが作られます。仕様の `ClassDefinitionEvaluation` は、`extends` の評価、二つのobjectの関連付け、class elementsの処理を行います。instance methodはprototypeへ定義され、instance fieldの定義情報は各instanceの初期化まで保持され、static fieldとstatic blockはclass評価中に実行されます。

class名 `User` の値はfunction objectです。

```js
console.log(typeof User); // function
console.log(Object.hasOwn(User, "prototype")); // true
console.log(User.prototype.constructor === User); // true
```

class constructorは `[[Construct]]` を持つためconstructできますが、通常関数としての呼び出しは拒否されます。

```js
// User("Taro"); // TypeError
```

この挙動はV8固有の都合ではありません。準拠実装に要求される動作です。

## 仕様層：methodとfieldは同時には作られない

class bodyの要素は、instance method、instance field、static elementで処理時点が違います。

通常のinstance methodはclass定義評価時にprototype objectへ定義されます。

```js
const descriptor = Object.getOwnPropertyDescriptor(
  User.prototype,
  "greet",
);

console.log(descriptor);
// value: function, writable: true,
// enumerable: false, configurable: true
```

一方、`role = "member"` はclass定義時に `User.prototype.role` を作りません。field definitionに対応する情報がconstructor側へ関連付けられ、各instanceの初期化時に評価されます。

```js
console.log(Object.hasOwn(User.prototype, "role")); // false

const first = new User("Taro");
const second = new User("Hanako");

console.log(Object.hasOwn(first, "role"));  // true
console.log(Object.hasOwn(second, "role")); // true
```

field initializerの式もinstanceごとに実行されます。題材の `crypto.randomUUID()` が毎回呼ばれるのはこのためです。methodのfunction objectをprototypeで共有することと、fieldの値を各instanceへ持つことを区別できます。

static fieldとstatic blockはclass定義評価の終盤で実行され、instance生成を待ちません。

```js
let count = 0;

class Example {
  static id = ++count;
  instanceId = ++count;
}

console.log(Example.id); // 1
console.log(count);      // 1

new Example();
console.log(count);      // 2
```

これも言語が定める評価タイミングです。V8がどのbytecodeへ変換するかにかかわらず、同じ順序が観測されなければなりません。

## 仕様層：newから[[Construct]]へ進む

`new User(" Taro ")` を評価すると、仕様はconstructorがconstruct可能か確認し、抽象操作 `Construct` を経由して `User.[[Construct]]` を呼びます。引数リストに加え、`NewTarget` として最初のconstructorも渡します。

通常の基底classの `[[Construct]]` は、要点を絞ると次の順序です。

1. constructor呼び出し用の新しい実行contextを準備する
2. `OrdinaryCreateFromConstructor(NewTarget, "%Object.prototype%")` で新しいordinary objectを作る
3. そのobjectをconstructor本体の `this` としてbindする
4. `InitializeInstanceElements` でprivate method/accessorとinstance fieldを初期化する
5. constructor bodyを評価する
6. 明示returnの規則に従い、最終的なobjectを返す

この順序から、基底classではfield initializerがconstructor bodyより先に実行されることが分かります。

```js
class Counter {
  value = 1;

  constructor() {
    console.log(this.value); // 1
    this.value = 2;
  }
}

console.log(new Counter().value); // 2
```

constructor body内の代入が先に実行され、その後field initializerが上書きするわけではありません。これは特定エンジンの都合ではなく仕様上の順序です。

## 仕様層：NewTargetがprototypeを決める

`OrdinaryCreateFromConstructor` は、`NewTarget.prototype` がオブジェクトなら、新しいordinary objectの `[[Prototype]]` に使います。オブジェクトでなければ、constructorのrealmにある既定intrinsic prototypeへフォールバックします。

継承時に基底constructorまで `NewTarget` が伝わるため、基底側でobjectを作っても最終的な派生prototypeが選ばれます。

```js
class Entity {
  constructor() {
    console.log(new.target.name);
  }
}

class UserEntity extends Entity {}

const entity = new UserEntity(); // UserEntity

console.log(
  Object.getPrototypeOf(entity) === UserEntity.prototype,
); // true
```

「`super()` は単に `Entity.call(this)` を実行する」という説明では、この動作を再現できません。class constructorは `call` できず、派生constructorでは `this` が `super()` の結果としてbindされるからです。

## 仕様層：派生classではsuper後にfieldを初期化する

派生constructorは、基底classと違って最初に自分の `this` objectを作りません。`super(...)` が基底constructorをconstructし、その結果を派生側の `this` としてbindします。その直後に、派生classのinstance elementsを初期化します。

```js
class Base {
  baseField = console.log("1: base field");

  constructor() {
    console.log("2: base constructor");
  }
}

class Derived extends Base {
  derivedField = console.log("3: derived field");

  constructor() {
    console.log("0: before super");
    super();
    console.log("4: derived constructor");
  }
}

new Derived();
```

出力順は `0, 1, 2, 3, 4` です。`super()` より前にログは出せますが、派生側の `this` を参照すると `ReferenceError` になります。

この順序は、基底constructorからoverride可能なmethodを呼ぶ危険も説明します。呼び出された派生methodは、まだ派生fieldが初期化されていない状態を観測する可能性があります。これは設計上避けるべきですが、原因の説明は仕様層だけで完結します。

## 仕様層：fieldは代入ではなくdefineされる

public field initializerは、見た目が `this.name = value` に近くても、仕様上は `DefineField` を通じてown data propertyを定義します。通常の `[[Set]]` と同じではありません。

この違いは、prototype上にsetterがある場合に観測できます。

```js
class Base {
  set value(next) {
    console.log("setter", next);
  }
}

class FieldExample extends Base {
  value = 1;
}

class AssignmentExample extends Base {
  constructor() {
    super();
    this.value = 1;
  }
}

const field = new FieldExample(); // setterは呼ばれない
new AssignmentExample();          // setter 1

console.log(Object.hasOwn(field, "value")); // true
```

field初期化はreceiver自身へpropertyをdefineするため、継承したsetterを呼びません。通常代入は `[[Set]]` を通り、setterを見つけて呼びます。

public fieldとして作られるpropertyは、通常 `writable: true`、`enumerable: true`、`configurable: true` です。

```js
console.log(
  Object.getOwnPropertyDescriptor(field, "value"),
);
```

transpilerの設定や古い変換方式によっては、class fieldを単純代入へ変換し、setterに関する挙動がnative実行と異なることがあります。TypeScriptのtargetや `useDefineForClassFields`、Babel設定を確認する理由になります。

## 仕様層：private elementは通常propertyではない

`#token` は文字列 `"#token"` やSymbol keyとして外部から取得できません。仕様はprivate nameとprivate element用の抽象操作を定義し、objectに対応するprivate elementがなければアクセス時に `TypeError` とします。

```js
class Wallet {
  #balance = 0;

  deposit(amount) {
    this.#balance += amount;
  }
}

const wallet = new Wallet();

console.log(Reflect.ownKeys(wallet)); // []
console.log(Object.hasOwn(wallet, "#balance")); // false
// wallet.#balance; // class外なのでSyntaxError
```

private method/accessorとprivate fieldは仕様上の追跡方法にも違いがありますが、利用コードは特定のメモリ配置を仮定できません。保証されるのは、通常のproperty reflectionに現れず、宣言classのprivate nameを持つコードだけがアクセスできることです。

## ここからV8実装層へ移る

ここまでの内容は、V8、SpiderMonkey、JavaScriptCoreなどが観測結果として満たすべきものです。ここから扱うMapやinline cacheはV8固有であり、他engineや将来versionが同じ表現を使う保証はありません。

| 層 | 用語の例 | 互換性上の位置付け |
| --- | --- | --- |
| ECMAScript仕様 | `[[Construct]]`、`NewTarget`、`InitializeInstanceElements`、`[[Prototype]]` | 準拠実装が観測可能な動作を守る |
| V8実装 | Map、DescriptorArray、transition、inline cache、Ignition、TurboFan | V8が変更でき、他engineは別方式を選べる |
| 標準API | `Object.getPrototypeOf`、property descriptor | portableな検証に使える |
| V8内部debug機能 | `%DebugPrint`、内部flag | 非標準でversion依存 |

仕様の角括弧付き内部slotは、同名propertyとして読めません。V8のC++ layoutやMap pointerも、ECMAScriptが要求する配置ではありません。

## V8実装層：Map（Hidden Class）でshapeを表す

公式資料では、V8の各heap objectはMapと呼ばれる内部構造への参照を持ちます。一般向けの説明ではHidden Classとも呼ばれますが、JavaScriptの `class` とは別物です。

Mapは、objectのshapeに関する情報を表します。named propertyの情報、property数、prototypeへの参照などが関連し、同じshapeを持つobjectがMapやdescriptor情報を共有できるようにします。

```js
function createPoint(x, y) {
  return { x, y };
}

const a = createPoint(1, 2);
const b = createPoint(3, 4);
```

同じ順序で同じnamed propertyを持つ `a` と `b` は、V8内部で同じMapへ到達しやすくなります。空objectへ `x` を追加し、次に `y` を追加するようなshape変化は、Map間のtransitionとして管理されます。

```text
Map0（propertyなし）
  └─ xを追加 → Map1（x）
                 └─ yを追加 → Map2（x, y）
```

一方、追加順が異なれば別のtransition pathになり得ます。

```js
const first = {};
first.x = 1;
first.y = 2;

const second = {};
second.y = 2;
second.x = 1;
```

最終的なown keyが同じでも、V8内部では異なるMapになる可能性があります。ただし、「このコードなら必ず同じMap」「propertyをこの順に書けば何ナノ秒速い」と仕様から断定はできません。V8のversion、実行feedback、propertyの種類、後続操作によって表現は変わります。

## V8実装層：propertyの保存場所は一種類ではない

V8では、named propertyとarray index相当のelementを別のstoreとして扱います。named propertyにも、object本体へ置くin-object property、別のproperties store、追加削除が多い場合のdictionary propertyなどがあります。

```js
const object = { name: "Taro" };
console.log(object.name); // Taro
```

保存形式に関係なく同じ参照結果を返す必要があります。JavaScript objectをC言語の固定structのように描く図は概念説明に限定し、offsetやbyte数をportableな事実として扱いません。

## V8実装層：field初期化とinline cache

class fieldはdefine semanticsを持ち、単純代入とは異なります。V8公式記事によると、過去にはfield初期化へruntime callを多く使い、その後、field定義向けbytecodeとinline cacheを導入して高速化しました。

inline cacheは、同じ操作箇所で過去に観測したMapなどのfeedbackを利用します。安定したshapeが繰り返されれば、field追加のtransitionを予測しやすくなります。

setterを呼ばないことなど、仕様の結果は最適化後も同じでなければなりません。最適化できなければ遅い経路へ戻れます。bytecode名やopcodeはversion依存なので、引用時は「その時点のV8実装」と明記します。

## constructorの書き方とshapeの安定性

instanceごとにpropertyの有無や追加順が変わると、V8が複数のshapeを扱う可能性が高まります。

```js
class User {
  constructor(name, isAdmin) {
    this.name = name;
    if (isAdmin) {
      this.permissions = ["all"];
    }
  }
}
```

この例では、adminだけが `permissions` を持つため複数shapeになります。ただし、不在に意味があるならこの設計が正しいです。性能のために意味を変えてはいけません。

意味上、全Userがpermissionsを持つなら、常に初期化する方がdomain modelもshapeも揃います。

```js
class User {
  constructor(name, isAdmin) {
    this.name = name;
    this.permissions = isAdmin ? ["all"] : [];
  }
}
```

これはまず不変条件とAPIの改善で、shapeの安定は副次的な利点です。property順を調整する前に現実のworkloadをprofilerで測ります。生成後の大量な追加削除やprototype変更も、最適化以前にobjectの契約を途中で変えるため避けます。

## allocationとGCを断定しない

ECMAScriptは、「`new` ごとに何byte確保する」「instanceをstackかheapのどちらへ置く」と定めません。V8ではobjectをgarbage-collected heapで管理しますが、JITが観測上不要なallocationを除去できる場合もあります。

したがって、「constructor一回で固定layoutの領域が必ず一個増える」「local変数なのでstackに置かれる」と断定しません。メモリ問題は、対象versionのDevToolsやNode.js inspectorでheap snapshot、allocation sampling、GCの状況を計測します。

## 標準APIとV8内部debug機能を使い分ける

仕様上の挙動を確認するだけなら、portableな標準APIで十分です。

```js
import assert from "node:assert/strict";

class Account {
  status = "active";

  constructor(name) {
    this.name = name;
  }

  close() {
    this.status = "closed";
  }
}

const account = new Account("Taro");

assert.equal(Object.getPrototypeOf(account), Account.prototype);
assert.deepEqual(Object.keys(account), ["status", "name"]);
assert.equal(Object.hasOwn(account, "close"), false);

const statusDescriptor = Object.getOwnPropertyDescriptor(
  account,
  "status",
);
assert.equal(statusDescriptor?.enumerable, true);
assert.equal(statusDescriptor?.writable, true);
assert.equal(statusDescriptor?.configurable, true);
```

Mapを観測するV8内部関数 `%DebugPrint` などは、d8や特定flag付き環境で使われる非標準debug機能です。構文自体が通常のJavaScriptとしてportableではありません。名称や出力も変更されます。production codeやlibraryの分岐条件に使いません。

性能を調べるときも、一回のconsole計測や内部Mapの見た目だけで結論を出しません。warm-upを含む現実的なworkload、複数回の計測、CPU profile、heap profileを使い、変更前後の利用者向け指標を比較します。

## まとめ

constructor内部は、ECMAScriptの保証を確認してからV8実装を重ねて読みます。

### ECMAScript仕様が説明すること

- class評価でconstructor function、prototype、method、field定義、static elementが準備される
- `new` は `[[Construct]]` を呼び、`NewTarget` がprototype選択へ関わる
- 基底fieldはconstructor body前、派生fieldは `super()` 後に初期化される
- public fieldはdefine semanticsを持ち、継承setterを呼ばない
- private elementなどの観測可能な規則も仕様が定める

### V8実装が説明すること

- MapやDescriptorArrayでshape情報を管理する
- propertyの種類に応じて複数の保存形式を使い分ける
- field定義やproperty accessをinline cacheなどで最適化する
- layoutやbytecodeはversionと実行状況で変わる

仕様用語は「何が起きるべきか」、V8用語は「一つのengineがどう実現するか」を答えます。この境界を保てば、実装詳細を言語保証として断定せずに内部最適化を学べます。

## 参考資料

- [ECMAScript仕様：Class Definitions](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-class-definitions)
- [ECMAScript仕様：ECMAScript Function Objects [[Construct]]](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-ecmascript-function-objects-construct-argumentslist-newtarget)
- [ECMAScript仕様：InitializeInstanceElements](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-initializeinstanceelements)
- [ECMAScript仕様：DefineField](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-definefield)
- [ECMAScript仕様：OrdinaryCreateFromConstructor](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinarycreatefromconstructor)
- [V8公式：Fast properties in V8](https://v8.dev/blog/fast-properties)
- [V8公式：Faster initialization of instances with new class features](https://v8.dev/blog/faster-class-features)
- [V8公式：Maps (Hidden Classes) in V8](https://v8.dev/docs/hidden-classes)
