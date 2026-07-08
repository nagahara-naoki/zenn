---
title: "JavaScriptにおける `this` の基本"
emoji: "🤖"
type: "tech"
topics: ["javascript"]
published: true
---
## 嫌われる理由・使いどころ・実務での注意点まで整理する

JavaScriptを学んでいると、多くの人が一度は `this` でつまずきます。

```js
console.log(this);
```

この `this` は、単純に「自分自身」を指すものではありません。

JavaScriptの `this` は、**関数がどのように呼び出されたかによって参照先が変わる特殊な値**です。

この記事では、JavaScriptにおける `this` の基本、嫌われるポイント、実務での使いどころ、そして安全な使い方を整理します。

---

# `this` とは

`this` とは、現在の実行コンテキストに紐づく特別な値です。

もう少し実務寄りに言うと、`this` は次のような場面で使われます。

- オブジェクト自身のプロパティを参照する
- クラスのインスタンス自身を参照する
- コンストラクター関数で生成中のオブジェクトを参照する
- イベント発生元の要素を参照する
- `call` / `apply` / `bind` で明示的に参照先を指定する

ただし、`this` は「書いた場所」だけで決まるわけではありません。

重要なのは、**呼び出し方**です。

---

# `this` が嫌われる理由

`this` が嫌われる一番の理由は、直感とズレやすいからです。

多くの人は、次のように考えがちです。

```txt
this = 今書いているオブジェクト自身
```

しかし、JavaScriptでは必ずしもそうではありません。

実際には、次のように考える必要があります。

```txt
this = 関数が呼び出されたときに決まる値
```

このルールが原因で、次のような問題が起きます。

- 関数を別の変数に代入すると `this` が変わる
- コールバック関数内で `this` が変わる
- アロー関数と通常関数で `this` の挙動が違う
- strict mode かどうかで挙動が変わる
- クラスメソッドを渡したときに `this` が外れる

`this` は便利ですが、扱いを間違えるとバグの原因になりやすいです。

---

# 1. 通常の関数での `this`

まず、通常の関数で `this` を確認します。

```js
function showThis() {
  console.log(this);
}

showThis();
```

この場合、ブラウザの非strict modeでは `this` はグローバルオブジェクトを指します。

ブラウザでは多くの場合、`window` です。

```txt
Window
```

ただし、strict modeでは `undefined` になります。

```js
"use strict";

function showThis() {
  console.log(this);
}

showThis();
```

```txt
undefined
```

この違いが、`this` をややこしくしているポイントのひとつです。

## 使いどころ

通常の関数で `this` に依存する書き方は、現在の実務ではあまりおすすめしません。

理由は、呼び出し方によって `this` が変わりやすいからです。

```js
function showName() {
  console.log(this.name);
}
```

このような関数は、単体で見ると `this` が何を指すのか分かりにくいです。

できるだけ引数で受け取る方が安全です。

```js
function showName(user) {
  console.log(user.name);
}
```

---

# 2. オブジェクトのメソッドでの `this`

`this` が分かりやすく使える代表例が、オブジェクトのメソッドです。

```js
const user = {
  name: "田中",
  greet() {
    console.log(`${this.name}さん、こんにちは`);
  },
};

user.greet();
```

出力結果です。

```txt
田中さん、こんにちは
```

この場合、`this` は `user` オブジェクトを指します。

```js
user.greet();
```

`user` から `greet` を呼び出しているため、`this` は `user` になります。

## 使いどころ

オブジェクトが持つデータに対して、そのオブジェクト自身の処理を書きたい場合に向いています。

```js
const cart = {
  items: [],
  addItem(item) {
    this.items.push(item);
  },
  getCount() {
    return this.items.length;
  },
};

cart.addItem("りんご");
cart.addItem("みかん");

console.log(cart.getCount());
```

このように、データと処理をひとまとめにしたいときに `this` は便利です。

---

# 3. メソッドを取り出すと `this` が外れる

`this` の代表的な落とし穴です。

```js
const user = {
  name: "田中",
  greet() {
    console.log(`${this.name}さん、こんにちは`);
  },
};

const greet = user.greet;

greet();
```

一見、`user.greet()` と同じように見えますが、これは同じではありません。

```js
user.greet();
```

この場合は、`user` から呼び出しています。

```js
greet();
```

この場合は、ただの関数として呼び出しています。

そのため、`this` は `user` ではなくなります。

## なぜ起きるのか

`this` は関数の定義場所ではなく、呼び出し方で決まるからです。

```js
const greet = user.greet;
```

この時点で、`greet` は `user` から切り離された関数になります。

そのため、呼び出したときに `this` が失われます。

## 対策

`bind` を使うと、`this` を固定できます。

```js
const greet = user.greet.bind(user);

greet();
```

```txt
田中さん、こんにちは
```

ただし、実務では `this` が外れるような設計を避けることも大切です。

---

# 4. アロー関数の `this`

アロー関数は、自分自身の `this` を持ちません。

```js
const user = {
  name: "田中",
  greet: () => {
    console.log(this.name);
  },
};

user.greet();
```

この場合、`this` は `user` を指しません。

アロー関数の `this` は、外側のスコープの `this` をそのまま使います。

## メソッドにアロー関数を使う注意点

オブジェクトのメソッドで `this` を使いたい場合、アロー関数は避けた方がよいです。

悪い例です。

```js
const user = {
  name: "田中",
  greet: () => {
    console.log(this.name);
  },
};
```

良い例です。

```js
const user = {
  name: "田中",
  greet() {
    console.log(this.name);
  },
};
```

オブジェクト自身を `this` で参照したい場合は、通常のメソッド構文を使うのが安全です。

## アロー関数が便利な場面

一方で、アロー関数が便利な場面もあります。

特に、内側の関数で外側の `this` を使いたいときです。

```js
const timer = {
  seconds: 0,
  start() {
    setInterval(() => {
      this.seconds++;
      console.log(this.seconds);
    }, 1000);
  },
};

timer.start();
```

この場合、`setInterval` の中をアロー関数にすることで、外側の `start` メソッドの `this` を使えます。

もし通常関数で書くと、`this` が変わってしまいます。

```js
const timer = {
  seconds: 0,
  start() {
    setInterval(function () {
      this.seconds++;
      console.log(this.seconds);
    }, 1000);
  },
};
```

このように、アロー関数は「`this` を持たない」ことが弱点でもあり、強みでもあります。

---

# 5. クラスでの `this`

クラスでは、`this` はインスタンス自身を指します。

```js
class User {
  constructor(name) {
    this.name = name;
  }

  greet() {
    console.log(`${this.name}さん、こんにちは`);
  }
}

const user = new User("田中");

user.greet();
```

出力結果です。

```txt
田中さん、こんにちは
```

この場合、`this.name` は生成された `user` インスタンスの `name` を指します。

## constructor内の `this`

`constructor` は、インスタンス生成時に実行される特別なメソッドです。

```js
class User {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
}
```

`new User("田中", 25)` を実行すると、新しいオブジェクトが作られ、そのオブジェクトに対して `this` が紐づきます。

```js
const user = new User("田中", 25);

console.log(user.name);
console.log(user.age);
```

## クラスメソッドを渡すと `this` が外れる

クラスでも、メソッドをそのまま渡すと `this` が外れることがあります。

```js
class Counter {
  constructor() {
    this.count = 0;
  }

  increment() {
    this.count++;
    console.log(this.count);
  }
}

const counter = new Counter();

const increment = counter.increment;

increment();
```

この場合、`increment` は `counter` から切り離されて呼ばれるため、`this` が期待通りになりません。

## 対策

`bind` で固定します。

```js
const increment = counter.increment.bind(counter);

increment();
```

または、呼び出し側で常にインスタンス経由にします。

```js
counter.increment();
```

---

# 6. DOMイベントでの `this`

DOMイベントでは、通常関数を使うと `this` がイベント発生元の要素を指す場合があります。

```js
const button = document.querySelector("button");

button.addEventListener("click", function () {
  console.log(this);
});
```

この場合、`this` はクリックされた `button` 要素です。

```js
button.addEventListener("click", function () {
  this.textContent = "クリックされました";
});
```

## アロー関数の場合

アロー関数を使うと、`this` はイベント発生元の要素を指しません。

```js
button.addEventListener("click", () => {
  console.log(this);
});
```

イベント対象を使いたい場合は、`event.currentTarget` を使うと明確です。

```js
button.addEventListener("click", (event) => {
  event.currentTarget.textContent = "クリックされました";
});
```

## 実務では `event.currentTarget` が安全

DOMイベントでは、`this` に頼るよりも `event.currentTarget` を使った方が読みやすいです。

```js
button.addEventListener("click", (event) => {
  const button = event.currentTarget;

  button.disabled = true;
});
```

`this` よりも、何を参照しているのかが明確になります。

---

# 7. `call` / `apply` / `bind`

JavaScriptでは、`this` を明示的に指定する方法があります。

それが `call`、`apply`、`bind` です。

## call

`call` は、`this` と引数を指定して関数を実行します。

```js
function greet(message) {
  console.log(`${message}、${this.name}さん`);
}

const user = {
  name: "田中",
};

greet.call(user, "こんにちは");
```

出力結果です。

```txt
こんにちは、田中さん
```

## apply

`apply` も `this` を指定して関数を実行します。

`call` との違いは、引数を配列で渡すことです。

```js
greet.apply(user, ["こんにちは"]);
```

## bind

`bind` は、`this` を固定した新しい関数を作ります。

```js
const greetUser = greet.bind(user);

greetUser("こんにちは");
```

`call` と `apply` はその場で実行します。

`bind` はあとで実行できる関数を作ります。

## 使い分け

| メソッド | 特徴 |
|---|---|
| `call` | `this` を指定してすぐ実行する |
| `apply` | `this` を指定してすぐ実行する。引数は配列 |
| `bind` | `this` を固定した関数を作る |

現在の実務では、`bind` はクラスメソッドやコールバックで `this` を固定したいときに使われることがあります。

---

# 8. `this` を使うべき場面

`this` は嫌われがちですが、使うべき場面もあります。

## オブジェクトのメソッド

```js
const user = {
  name: "田中",
  greet() {
    console.log(`${this.name}さん、こんにちは`);
  },
};
```

オブジェクト自身のデータを使う処理では、`this` が自然です。

## クラス

```js
class User {
  constructor(name) {
    this.name = name;
  }

  greet() {
    console.log(`${this.name}さん、こんにちは`);
  }
}
```

クラスでは、インスタンスの状態を扱うために `this` を使います。

## コンストラクター関数

```js
function User(name) {
  this.name = name;
}

const user = new User("田中");
```

古いJavaScriptでは、コンストラクター関数で `this` がよく使われていました。

現在は `class` を使うことが多いですが、古いコードを読むためには理解しておく必要があります。

---

# 9. `this` を避けた方がよい場面

`this` は便利ですが、無理に使う必要はありません。

## 単純な関数

悪い例です。

```js
function getName() {
  return this.name;
}
```

この関数は、`this` が何を指すか分かりにくいです。

良い例です。

```js
function getName(user) {
  return user.name;
}
```

引数で受け取った方が明確です。

## コールバック関数

コールバック内で `this` に依存すると、意図しない挙動になりやすいです。

```js
setTimeout(function () {
  console.log(this.name);
}, 1000);
```

必要であれば、アロー関数を使うか、引数で値を渡す方が安全です。

```js
setTimeout(() => {
  console.log(user.name);
}, 1000);
```

## 配列メソッド

`map` や `filter` では、基本的に `this` に頼らない方が読みやすいです。

```js
const names = users.map((user) => user.name);
```

このように、引数で受け取った値を使う方が明確です。

---

# 10. `this` の覚え方

`this` は、次の順番で考えると整理しやすいです。

| 呼び出し方 | `this` が指すもの |
|---|---|
| `obj.method()` | `obj` |
| `func()` | strict modeでは `undefined` |
| `new Func()` | 新しく作られるオブジェクト |
| `func.call(obj)` | `obj` |
| `func.apply(obj)` | `obj` |
| `func.bind(obj)` | `obj` に固定された関数 |
| アロー関数 | 外側の `this` |

特に重要なのは、次の2つです。

```txt
this は定義場所ではなく、呼び出し方で決まる
アロー関数は自分自身の this を持たない
```

この2つを押さえると、`this` の混乱はかなり減ります。

---

# 11. 実務でのおすすめ方針

実務では、次の方針にすると `this` によるバグを減らせます。

## 基本は引数で渡す

```js
function formatUserName(user) {
  return `${user.lastName} ${user.firstName}`;
}
```

単純な処理では、`this` より引数の方が分かりやすいです。

## オブジェクトやクラスでは `this` を使う

```js
class Cart {
  constructor() {
    this.items = [];
  }

  addItem(item) {
    this.items.push(item);
  }
}
```

状態を持つオブジェクトでは、`this` が自然です。

## メソッドでアロー関数を使いすぎない

```js
const user = {
  name: "田中",
  greet() {
    console.log(this.name);
  },
};
```

`this` を使うメソッドは、通常のメソッド構文で書くのが安全です。

## コールバックではアロー関数を活用する

```js
class Timer {
  constructor() {
    this.seconds = 0;
  }

  start() {
    setInterval(() => {
      this.seconds++;
    }, 1000);
  }
}
```

外側の `this` を使いたいコールバックでは、アロー関数が便利です。

## DOMイベントでは `event.currentTarget` を使う

```js
button.addEventListener("click", (event) => {
  event.currentTarget.disabled = true;
});
```

`this` よりも参照先が明確になります。

---

# 12. よくあるミス

## ミス1：メソッドを変数に入れて `this` が外れる

```js
const method = user.greet;

method();
```

対策です。

```js
const method = user.greet.bind(user);
```

## ミス2：オブジェクトメソッドをアロー関数で書く

```js
const user = {
  name: "田中",
  greet: () => {
    console.log(this.name);
  },
};
```

対策です。

```js
const user = {
  name: "田中",
  greet() {
    console.log(this.name);
  },
};
```

## ミス3：コールバック内の `this` が変わる

```js
setTimeout(function () {
  console.log(this.name);
}, 1000);
```

対策です。

```js
setTimeout(() => {
  console.log(this.name);
}, 1000);
```

ただし、外側の `this` が何を指しているかは確認が必要です。

## ミス4：イベントでアロー関数の `this` を使う

```js
button.addEventListener("click", () => {
  this.disabled = true;
});
```

対策です。

```js
button.addEventListener("click", (event) => {
  event.currentTarget.disabled = true;
});
```

---

# まとめ

JavaScriptの `this` は、便利ですが混乱しやすい仕組みです。

特に大切なのは、次の2点です。

```txt
this は呼び出し方で決まる
アロー関数は自分自身の this を持たない
```

`this` は、オブジェクトのメソッドやクラスの中では自然に使えます。

一方で、単純な関数やコールバック、配列メソッドでは、無理に `this` を使わず、引数で値を渡した方が読みやすいことが多いです。

実務では、次のように考えると安全です。

| 場面 | おすすめ |
|---|---|
| 通常の関数 | 引数で受け取る |
| オブジェクトメソッド | `this` を使ってよい |
| クラス | `this` を使う |
| コールバック | アロー関数を検討する |
| DOMイベント | `event.currentTarget` を使う |
| メソッドを渡す | `bind` を検討する |

`this` は完全に避けるものではありません。

ただし、どこでも使うものでもありません。

「その `this` は何を指しているのか」を説明できる場面でだけ使うと、バグが少なく読みやすいコードになります。
