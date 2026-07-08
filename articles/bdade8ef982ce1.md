---
title: "constructor の内部構造を理解する"
emoji: "🤖"
type: "tech"
topics: ["javascript"]
published: true
---

JavaScript / TypeScript の `constructor` は、クラスからインスタンスを作るときに最初に実行される初期化処理です。

たとえば、次のようなコードがあります。

```ts
class User {
  name: string

  constructor(name: string) {
    this.name = name
  }
}

const user = new User("Taro")
```

表面的には、`new User("Taro")` によって `constructor` が呼ばれ、`this.name` に `"Taro"` が入るだけに見えます。

しかし、内部ではもう少し多くのことが行われています。

この記事では、`constructor` を単なる「初期化用メソッド」としてではなく、JavaScript エンジン内部でどのように扱われているのかという視点で整理します。

---

## この記事で理解すること

この記事では、次の内容を扱います。

- `constructor` は内部的に何者なのか
- `new` を実行したときに何が起きるのか
- `this` はいつ、どのように作られるのか
- `prototype` はどこで使われるのか
- `this.name = name` のような代入は内部で何をしているのか
- class の constructor と通常関数の constructor は何が違うのか
- 継承時の `super()` はなぜ必要なのか
- V8 ではオブジェクトがどのように管理されるのか

---

## まず結論

JavaScript の `constructor` は、内部的には **`new` されたときに実行される特別な関数オブジェクト** です。

もう少し正確に言うと、`new User()` のように呼び出されたとき、JavaScript エンジンは `User` という関数オブジェクトが持つ内部処理である `[[Construct]]` を実行します。

イメージとしては、次のような流れです。

```txt
class User { ... }
        ↓
User という関数オブジェクトが作られる
        ↓
User は [[Construct]] という内部処理を持つ
        ↓
new User() される
        ↓
User.[[Construct]] が実行される
        ↓
新しいオブジェクトが作られる
        ↓
User.prototype とつながる
        ↓
constructor 本体が this 付きで実行される
        ↓
完成したインスタンスが返る
```

つまり `constructor` の正体は、単にクラスの中にある普通のメソッドではありません。

`new` 演算子と連動して、オブジェクト生成処理の入口になる特別な関数です。

---

## constructor は「関数オブジェクト」

次のコードを見てみます。

```ts
class User {
  constructor(name: string) {
    this.name = name
  }

  greet() {
    console.log("Hello")
  }
}
```

この `class User` は、見た目はクラス構文ですが、内部的には `User` という関数オブジェクトが作られます。

イメージとしてはこうです。

```txt
User
├─ 関数オブジェクト
├─ [[Call]]
├─ [[Construct]]
├─ [[ECMAScriptCode]] = constructor の本体
├─ [[ConstructorKind]] = base
├─ [[IsClassConstructor]] = true
└─ prototype ──> User.prototype

User.prototype
├─ constructor ──> User
└─ greet ──> function
```

ここで重要なのは、`constructor` の本体は `User` 関数オブジェクト側にあるということです。

一方、`greet` のような通常メソッドは `User.prototype` 側に置かれます。

---

## constructor と通常メソッドの置き場所

次のクラスを考えます。

```ts
class User {
  constructor(name: string) {
    this.name = name
  }

  greet() {
    console.log(`Hello, ${this.name}`)
  }
}
```

内部的な配置は、ざっくり次のようになります。

```txt
User 関数オブジェクト
└─ constructor の処理本体

User.prototype
├─ constructor ──> User
└─ greet ──> function

user インスタンス
├─ name: "Taro"
└─ [[Prototype]] ──> User.prototype
```

つまり、役割ごとに置かれる場所が違います。

| 要素 | 内部的な置き場所 |
|---|---|
| `constructor` 本体 | `User` 関数オブジェクト側 |
| `greet` メソッド | `User.prototype` 側 |
| `name` プロパティ | `new User()` で作られたインスタンス側 |

`constructor` は `User.prototype` に普通のメソッドとして保存されているわけではありません。

`User.prototype.constructor` というプロパティは存在しますが、これは「この prototype から作られたインスタンスの constructor は User です」という参照のようなものです。

---

## new User("Taro") の内部処理

次のコードを例にします。

```ts
class User {
  constructor(name: string) {
    this.name = name
  }
}

const user = new User("Taro")
```

この `new User("Taro")` の内部では、大きく次の処理が行われます。

```txt
1. User が constructor として使えるか確認する
2. User.[[Construct]] を呼び出す
3. 新しいオブジェクトを作る
4. そのオブジェクトの [[Prototype]] に User.prototype を設定する
5. constructor 本体を実行する
6. constructor 内の this に新しいオブジェクトを割り当てる
7. this.name = "Taro" を実行する
8. 完成したオブジェクトを返す
```

擬似コードで書くと、次のようなイメージです。

```js
function internalNew(Constructor, args) {
  // 1. Constructor.prototype を取得する
  const prototype = Constructor.prototype

  // 2. prototype とつながった新しいオブジェクトを作る
  const obj = createObjectLinkedTo(prototype)

  // 3. constructor 本体を obj を this として実行する
  const result = Constructor.apply(obj, args)

  // 4. constructor がオブジェクトを明示的に返した場合はそれを使う
  if (isObject(result)) {
    return result
  }

  // 5. 通常は作成済みの obj を返す
  return obj
}
```

実際の JavaScript エンジンはこの通りに実装されているわけではありません。

しかし、理解としてはこの流れでかなり近いです。

---

## [[Construct]] とは何か

JavaScript の仕様には、`[[Construct]]` という内部メソッドがあります。

これは、ある関数オブジェクトが `new` で呼び出されたときに実行される内部処理です。

たとえば、次のコードがあります。

```ts
const user = new User("Taro")
```

これは内部的には、かなり単純化すると次のようなイメージになります。

```txt
User.[[Construct]](["Taro"], User)
```

`[[Construct]]` は通常の JavaScript コードから直接呼ぶことはできません。

つまり、次のようには書けません。

```js
User.[[Construct]](["Taro"])
```

`[[Construct]]` は、JavaScript エンジン内部だけが扱う仕組みです。

---

## [[Call]] と [[Construct]] の違い

関数には、大きく分けて次の2つの呼ばれ方があります。

```js
User("Taro")      // 通常呼び出し
new User("Taro")  // constructor 呼び出し
```

内部的には、それぞれ別の処理です。

| 呼び方 | 内部処理 |
|---|---|
| `User("Taro")` | `[[Call]]` |
| `new User("Taro")` | `[[Construct]]` |

普通の関数は、`[[Call]]` と `[[Construct]]` の両方を持つことがあります。

```js
function User(name) {
  this.name = name
}

User("Taro")
new User("Taro")
```

このような通常関数は、普通に呼ぶことも、`new` で呼ぶこともできます。

しかし、class の constructor は少し違います。

```ts
class User {
  constructor(name: string) {
    this.name = name
  }
}

User("Taro") // TypeError
```

class は `new` なしで呼び出せません。

これは、class constructor には内部的に `[[IsClassConstructor]]` のようなフラグがあり、通常呼び出しされた場合にエラーにするためです。

イメージとしては、次のような判定が入ります。

```js
function internalCall(fn, thisArg, args) {
  if (fn.isClassConstructor) {
    throw new TypeError("Class constructor cannot be invoked without 'new'")
  }

  return execute(fn.code, thisArg, args)
}
```

つまり、class の constructor は関数オブジェクトではありますが、通常関数のように自由に呼べるわけではありません。

---

## this はいつ作られるのか

constructor を理解するうえで重要なのが `this` です。

```ts
class User {
  constructor(name: string) {
    this.name = name
  }
}
```

この `this` は、今まさに作られているインスタンスを指します。

`new User("Taro")` の内部では、constructor 本体が実行される前に、新しいオブジェクトが作られます。

```txt
new User("Taro")
        ↓
空のオブジェクトを作る
        ↓
そのオブジェクトを this として constructor を実行する
        ↓
this.name = "Taro"
        ↓
完成したオブジェクトを返す
```

つまり、constructor 内の `this.name = name` は、イメージとしては次のような処理です。

```js
const thisObj = {}
thisObj.name = "Taro"
return thisObj
```

ただし実際には、ただの `{}` ではなく、`User.prototype` とつながったオブジェクトが作られます。

---

## prototype はどこで使われるのか

次のコードを見てみます。

```ts
class User {
  constructor(name: string) {
    this.name = name
  }

  greet() {
    console.log("Hello")
  }
}

const user = new User("Taro")
```

このとき作られる `user` は、`User.prototype` とつながっています。

```txt
user
├─ name: "Taro"
└─ [[Prototype]] ──> User.prototype

User.prototype
├─ constructor ──> User
└─ greet ──> function
```

そのため、次のように `greet` を呼べます。

```ts
user.greet()
```

`user` 自身には `greet` は存在しません。

しかし、JavaScript はプロパティを探すときに、まず自分自身を見て、なければ `[[Prototype]]` をたどります。

```txt
user.greet を探す
        ↓
user 自身を見る
        ↓
greet はない
        ↓
user.[[Prototype]] を見る
        ↓
User.prototype.greet が見つかる
        ↓
実行する
```

このように、`new` の内部では `User.prototype` を使って、作成されるインスタンスの `[[Prototype]]` を設定しています。

---

## user.constructor はどこから来るのか

よく次のようなコードを見ます。

```ts
console.log(user.constructor === User) // true
```

これを見ると、`user` の中に `constructor` というプロパティが入っているように見えるかもしれません。

しかし、多くの場合、`user` 自身には `constructor` はありません。

実際には次のように探索されます。

```txt
user.constructor を探す
        ↓
user 自身を見る
        ↓
constructor はない
        ↓
User.prototype を見る
        ↓
User.prototype.constructor が見つかる
        ↓
User を返す
```

つまり、`user.constructor` はプロトタイプチェーン経由で見つかっているだけです。

`constructor` 本体がインスタンスの中にコピーされているわけではありません。

---

## constructor が値を return した場合

通常、constructor は明示的に値を返しません。

```ts
class User {
  constructor(name: string) {
    this.name = name
  }
}
```

この場合、`new User("Taro")` は自動的に `this` を返します。

ただし、constructor の中でオブジェクトを明示的に返すこともできます。

```js
class User {
  constructor(name) {
    this.name = name

    return {
      name: "Other"
    }
  }
}

const user = new User("Taro")
console.log(user.name) // "Other"
```

constructor がオブジェクトを返した場合、そのオブジェクトが `new` の結果になります。

一方で、プリミティブ値を返した場合は、通常は無視されます。

```js
class User {
  constructor(name) {
    this.name = name
    return 123
  }
}

const user = new User("Taro")
console.log(user.name) // "Taro"
```

このように、constructor の戻り値には特別なルールがあります。

ただし、実務では constructor から別のオブジェクトを返す書き方はあまり使いません。

可読性が下がりやすいため、基本的には `this` を初期化する場所として使うのが自然です。

---

## V8 ではオブジェクトをどう管理しているのか

ここからは、V8 のような JavaScript エンジン寄りの話です。

JavaScript のオブジェクトは、単純な連想配列としてだけ管理されているわけではありません。

たとえば、次のようなオブジェクトがあります。

```ts
const user = {
  name: "Taro",
  age: 20
}
```

このオブジェクトを、内部で毎回ただのハッシュマップとして扱うと遅くなります。

そこで V8 では、オブジェクトの「形」を表す情報を持ちます。

この「形」は、よく **HiddenClass** や **Map** と呼ばれます。

ざっくりした構造は次のようなイメージです。

```txt
JSObject
├─ Map / HiddenClass への参照
├─ properties
├─ elements
└─ in-object properties
```

C言語風の疑似構造で書くと、次のようなイメージです。

```c
typedef struct JSObject {
    Map* map;               // オブジェクトの形を表す
    FixedArray* properties;  // 通常プロパティ領域
    FixedArray* elements;    // 配列 index 系の要素
    JSValue fields[];        // オブジェクト内に直接持つ値
} JSObject;
```

もちろん実際の V8 の実装はもっと複雑です。

ここでは理解しやすいように、かなり簡略化しています。

---

## this.name = name の内部

constructor の中で、次のような処理を書いたとします。

```ts
class User {
  constructor(name: string) {
    this.name = name
  }
}
```

この `this.name = name` は、内部的には単に「オブジェクトにキーと値を追加する」だけではありません。

V8 のようなエンジンでは、おおまかに次のようなことが起こります。

```txt
1. this が持っている現在の Map を見る
2. name というプロパティを追加したときの遷移先 Map があるか確認する
3. なければ新しい Map を作る
4. this の Map を新しい Map に差し替える
5. name の値を決められたスロットに保存する
```

イメージとしてはこうです。

```txt
初期状態

user
├─ map ──> Map0
└─ fields: []

this.name = "Taro" 実行後

user
├─ map ──> Map1
└─ fields[0] = "Taro"

Map1
└─ name は fields[0] にある
```

さらに、次のように複数のプロパティを代入する場合を考えます。

```ts
class User {
  constructor(name: string, age: number) {
    this.name = name
    this.age = age
  }
}
```

この場合、Map は次のように遷移します。

```txt
Map0
  └─ add "name" ──> Map1
                      └─ add "age" ──> Map2
```

同じ順番でプロパティを追加するインスタンスは、同じ Map の遷移をたどれます。

そのため、JavaScript エンジンはオブジェクトの形を予測しやすくなり、最適化しやすくなります。

---

## constructor でプロパティの順番を揃える意味

次の2つのクラスを比べます。

```ts
class User {
  constructor(name: string, age: number) {
    this.name = name
    this.age = age
  }
}
```

こちらは、常に `name` → `age` の順でプロパティを追加します。

一方、次のように条件によって追加順が変わるコードがあります。

```ts
class User {
  constructor(name: string, age: number, isAdmin: boolean) {
    if (isAdmin) {
      this.role = "admin"
    }

    this.name = name
    this.age = age
  }
}
```

この場合、`isAdmin` の値によってオブジェクトの形が変わります。

```txt
isAdmin = true
Map0 → role → name → age

isAdmin = false
Map0 → name → age
```

つまり、同じ `User` から作られたインスタンスでも、内部的な Map が分かれやすくなります。

エンジンはこれでも動作できますが、同じ形のオブジェクトが揃っているほうが最適化しやすいです。

そのため、実務では constructor の中で、できるだけ同じプロパティを同じ順番で初期化するのが望ましいです。

```ts
class User {
  constructor(name: string, age: number, isAdmin: boolean) {
    this.name = name
    this.age = age
    this.role = isAdmin ? "admin" : "user"
  }
}
```

このようにすると、常に `name` → `age` → `role` の順番でプロパティが追加されます。

---

## class fields はどう扱われるのか

TypeScript / JavaScript では、constructor の外にフィールドを書くこともできます。

```ts
class Todo {
  completed = false
  createdAt = new Date()

  constructor(title: string) {
    this.title = title
  }
}
```

この場合、`completed` や `createdAt` もインスタンスごとに初期化されます。

内部的には、インスタンス作成時にフィールド初期化処理が実行され、その後 constructor 本体の処理が実行される、というイメージです。

単純化すると、次のような流れです。

```txt
new Todo("買い物")
        ↓
新しいオブジェクトを作る
        ↓
class field を初期化する
        ↓
completed = false
createdAt = new Date()
        ↓
constructor 本体を実行する
        ↓
title = "買い物"
        ↓
完成したインスタンスを返す
```

実際には、base class か derived class かによって初期化タイミングに違いがあります。

ただ、まずは「class fields も constructor 周辺のインスタンス初期化処理として扱われる」と理解するとよいです。

---

## 継承時の constructor はなぜ特殊なのか

次のコードを見てみます。

```ts
class Animal {
  name: string

  constructor(name: string) {
    this.name = name
  }
}

class Dog extends Animal {
  breed: string

  constructor(name: string, breed: string) {
    super(name)
    this.breed = breed
  }
}

const dog = new Dog("Pochi", "Shiba")
```

継承がない通常のクラスでは、constructor が開始される前に `this` になるオブジェクトが作られます。

```txt
base constructor
        ↓
先に this を作る
        ↓
constructor 本体を実行する
```

しかし、継承している子クラスでは、少し違います。

```txt
derived constructor
        ↓
Dog constructor 開始
        ↓
まだ this は使えない
        ↓
super(name) を呼ぶ
        ↓
親クラス Animal の constructor が実行される
        ↓
this が初期化される
        ↓
Dog 側の this.breed = breed が実行できる
```

そのため、次のコードはエラーになります。

```ts
class Dog extends Animal {
  constructor(name: string, breed: string) {
    this.breed = breed // エラー
    super(name)
  }
}
```

`super()` より前に `this` を使っているためです。

---

## derived constructor では this が未初期化

内部的には、継承しているクラスの constructor は `[[ConstructorKind]] = derived` のように扱われます。

通常の base constructor では、自分で `this` 用のオブジェクトを作れます。

```txt
new User()
        ↓
User constructor が this を作る
        ↓
constructor 本体を実行する
```

一方、derived constructor では、親クラスの constructor によって `this` が作られます。

```txt
new Dog()
        ↓
Dog constructor 開始
        ↓
this は未初期化
        ↓
super(name)
        ↓
Animal.[[Construct]] が呼ばれる
        ↓
Animal 側で this が作られる
        ↓
Dog の this として使えるようになる
```

だから、子クラスでは `super()` を呼ぶ前に `this` を使えません。

これは単なる文法ルールではなく、内部的に `this` がまだ存在していないためです。

---

## C言語風に見る constructor の内部構造

JavaScript エンジン内部を、かなり単純化して C言語風に表すと、次のような構造をイメージできます。

```c
typedef struct Map {
    DescriptorArray* descriptors;
    TransitionArray* transitions;
    JSObject* prototype;
    int instance_size;
} Map;

typedef struct JSObject {
    Map* map;
    FixedArray* properties;
    FixedArray* elements;
    JSValue in_object_fields[];
} JSObject;

typedef struct JSFunction {
    JSObject object;

    Code* code;                 // constructor 本体のコード
    Environment* environment;   // クロージャ環境
    JSObject* prototype_object; // User.prototype
    Map* initial_map;           // インスタンス生成時の初期 Map

    bool has_call;
    bool has_construct;
    bool is_class_constructor;
    ConstructorKind constructor_kind;
} JSFunction;
```

実際の V8 の構造とは異なりますが、考え方としては次のようになります。

- `JSFunction` が `User` という関数オブジェクトを表す
- `code` に constructor 本体がある
- `prototype_object` が `User.prototype` を指す
- `initial_map` が新規インスタンス用の初期形を表す
- `has_construct` が `new` 可能かどうかを表す
- `is_class_constructor` が class constructor かどうかを表す
- `constructor_kind` が base / derived の違いを表す

---

## new の処理を C風の擬似コードで見る

`new User("Taro")` の処理を、かなり単純化して C風に書くと次のようになります。

```c
JSObject* JS_New(JSFunction* ctor, JSValue* args) {
    if (!ctor->has_construct) {
        throw_type_error();
    }

    JSObject* prototype = ctor->prototype_object;

    JSObject* obj = AllocateJSObject(ctor->initial_map);
    SetPrototype(obj, prototype);

    JSValue result = ExecuteConstructorBody(ctor->code, obj, args);

    if (IsObject(result)) {
        return AsObject(result);
    }

    return obj;
}
```

そして constructor 内の、

```ts
this.name = name
```

は、単純化すると次のような処理です。

```c
void SetProperty(JSObject* obj, String* key, JSValue value) {
    Map* current = obj->map;

    int index;
    Map* next = FindTransition(current, key, &index);

    if (next == NULL) {
        next = CreateNewMapWithProperty(current, key, &index);
    }

    obj->map = next;
    obj->in_object_fields[index] = value;
}
```

これが、`this.name = "Taro"` の内部イメージです。

もちろん、実際のエンジンではもっと多くの処理があります。

たとえば、次のような要素も関わります。

- ガベージコレクション
- JIT コンパイル
- Inline Cache
- private field
- accessor property
- property descriptor
- dictionary mode
- realm
- strict mode
- super 呼び出し

この記事では、constructor の理解に必要な中心部分だけを取り出しています。

---

## 通常関数 constructor と class constructor の違い

JavaScript では、昔から次のような書き方がありました。

```js
function User(name) {
  this.name = name
}

const user = new User("Taro")
```

これは function constructor と呼ばれることがあります。

この場合、`User` は通常関数なので、次のようにも呼べます。

```js
User("Taro")
```

ただし、`new` なしで呼ぶと `this` の扱いが変わるため、意図しないバグにつながることがあります。

一方、class constructor は `new` なしでは呼べません。

```ts
class User {
  constructor(name: string) {
    this.name = name
  }
}

User("Taro") // TypeError
```

これは class が、より安全にオブジェクト生成専用として使えるように設計されているためです。

整理すると、次のようになります。

| 項目 | function constructor | class constructor |
|---|---|---|
| `new` で呼べる | 可能 | 可能 |
| 普通に呼べる | 可能 | 不可 |
| prototype を持つ | 持つ | 持つ |
| メソッド定義 | `User.prototype.xxx = ...` | class 構文内に書ける |
| `new` なしの誤用 | 起きやすい | 防げる |

内部的にはどちらも `[[Construct]]` を使ってインスタンスを作ります。

ただし、class constructor には通常呼び出しを禁止するための仕組みが入っています。

---

## constructor の内部処理まとめ

最後に、`constructor` の内部処理をもう一度整理します。

次のコードがあります。

```ts
class User {
  constructor(name: string, age: number) {
    this.name = name
    this.age = age
  }
}

const user = new User("Taro", 20)
```

このとき、内部では次のような流れになります。

```txt
1. User という関数オブジェクトを確認する
2. User が [[Construct]] を持っているか確認する
3. new User("Taro", 20) によって [[Construct]] を実行する
4. User.prototype を取得する
5. User.prototype を [[Prototype]] に持つ JSObject を作る
6. User 用の初期 Map / HiddenClass を設定する
7. constructor 本体を this 付きで実行する
8. this.name = "Taro" によって Map 遷移と値の保存が起きる
9. this.age = 20 によってさらに Map 遷移と値の保存が起きる
10. constructor がオブジェクトを返していなければ this を返す
11. user に完成したインスタンスが入る
```

完成後のイメージはこうです。

```txt
user
├─ map ──> User 用の Map
├─ name: "Taro"
├─ age: 20
└─ [[Prototype]] ──> User.prototype

User.prototype
└─ constructor ──> User
```

---

## 最重要ポイント

`constructor` は、単なる「初期化用メソッド」ではありません。

内部的には、`new` 演算子から呼び出される `[[Construct]]` を持った関数オブジェクトです。

そして、`new User()` の裏側では次の処理が行われています。

1. constructor 関数オブジェクトを確認する
2. `[[Construct]]` を呼び出す
3. 新しいオブジェクトを確保する
4. `User.prototype` をインスタンスの `[[Prototype]]` に設定する
5. constructor 本体を `this` 付きで実行する
6. `this.xxx = ...` によってプロパティを書き込む
7. V8 では HiddenClass / Map の遷移が起きる
8. 完成したオブジェクトを返す

つまり、constructor は次のように説明できます。

> constructor とは、`new` 演算子によって呼び出される `[[Construct]]` を持った関数オブジェクトであり、インスタンス生成、prototype 接続、this の束縛、プロパティ初期化を行う入口である。

この視点で見ると、`constructor`、`new`、`this`、`prototype`、`class`、`継承` がすべてつながって理解しやすくなります。
