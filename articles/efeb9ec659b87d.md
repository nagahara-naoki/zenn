---
title: "JavaScriptのthisを「呼び出し方」から理解する"
emoji: "🤖"
type: "tech"
topics: ["javascript", "this", "初心者"]
published: true
---

JavaScriptの `this` は、関数を書いた場所だけでは決まりません。通常の関数では、主に**どう呼び出したか**で決まります。

まず次の対応を押さえると、個別ルールを暗記せずに読めます。

| 呼び出し方 | `this` |
| --- | --- |
| `obj.method()` | `obj` |
| `func()` | strict modeでは `undefined` |
| `func.call(obj)` | `obj` |
| `new Func()` | 新しく生成されるオブジェクト |
| アロー関数 | 外側のレキシカルな `this` |

## メソッド呼び出し

```js
const user = {
  name: "Taro",
  greet() {
    return `Hello, ${this.name}`;
  },
};

console.log(user.greet()); // Hello, Taro
```

`user.greet()` と呼んでいるため、`greet` 内の `this` は `user` です。

同じ関数でも、別オブジェクトのプロパティとして呼ぶと `this` は変わります。

```js
const anotherUser = {
  name: "Hanako",
  greet: user.greet,
};

console.log(anotherUser.greet()); // Hello, Hanako
```

関数が `user` を記憶しているのではなく、呼び出し時のreceiverが使われています。

## メソッドを取り出すとthisが外れる

```js
const greet = user.greet;
greet();
```

この呼び出しには `user.` がありません。ES Modulesやstrict modeでは、通常関数として呼ばれた `greet` の `this` は `undefined` です。

コールバックとして渡すときも同じ問題が起こります。

```js
setTimeout(user.greet, 100);
```

### 対策1：呼び出しを包む

```js
setTimeout(() => {
  console.log(user.greet());
}, 100);
```

### 対策2：bindする

```js
const boundGreet = user.greet.bind(user);
setTimeout(boundGreet, 100);
```

`bind` は、指定した `this` を持つ新しい関数を返します。イベントの解除に同じ関数参照が必要な場合は、生成した関数を変数へ保存します。

## 通常関数を単独で呼ぶ

```js
"use strict";

function showThis() {
  console.log(this);
}

showThis(); // undefined
```

古い非strictのscriptをブラウザーで実行した場合は、グローバルオブジェクトへ置き換えられることがあります。

ただしES Modulesは常にstrict modeです。Node.jsのCommonJS、ブラウザーのclassic script、ES Modulesを混ぜて「通常関数の `this` はwindow」と覚えないようにします。

## アロー関数のthis

アロー関数には独自の `this` bindingがありません。外側のスコープから `this` を参照します。

```js
const counter = {
  count: 0,
  start() {
    setInterval(() => {
      this.count += 1;
      console.log(this.count);
    }, 1_000);
  },
};
```

アロー関数の外側にある `start` は `counter.start()` と呼ばれるため、その `this` は `counter` です。内側のアロー関数も同じ `this` を参照します。

一方、オブジェクトのメソッドそのものをアロー関数にすると、期待した動きになりません。

```js
const user = {
  name: "Taro",
  greet: () => {
    return this.name;
  },
};
```

アロー関数は `user` を `this` として受け取りません。オブジェクト自身を使うメソッドは、メソッド構文で書きます。

## classでのthis

```ts
class User {
  constructor(public readonly name: string) {}

  greet(): string {
    return `Hello, ${this.name}`;
  }
}

const user = new User("Taro");
console.log(user.greet());
```

classメソッドも、取り出して単独で呼ぶと `this` が失われます。

```ts
const greet = user.greet;
// greet(); // TypeError
```

UIフレームワークなどでメソッドを頻繁にコールバックへ渡す場合、クラスフィールドのアロー関数を選ぶことがあります。

```ts
class User {
  constructor(public readonly name: string) {}

  greet = (): string => {
    return `Hello, ${this.name}`;
  };
}
```

この関数はインスタンスごとに作られます。prototypeメソッドとの違いを理解したうえで選びます。

## call・apply・bind

通常の関数は、`call` と `apply` で `this` を明示して呼べます。

```js
function introduce(greeting, punctuation) {
  return `${greeting}, ${this.name}${punctuation}`;
}

const user = { name: "Taro" };

introduce.call(user, "Hello", "!");
introduce.apply(user, ["Hello", "!"]);
```

- `call`：引数を1つずつ渡す
- `apply`：引数を配列形式で渡す
- `bind`：呼び出さず、新しい関数を返す

アロー関数の `this` は `call`、`apply`、`bind` でも変更できません。

## DOMイベント

通常関数のevent listenerでは、ブラウザーが `this` を `currentTarget` と同じ値で呼び出します。

```js
button.addEventListener("click", function (event) {
  console.log(this === event.currentTarget); // true
});
```

アロー関数では外側の `this` を使います。

```js
button.addEventListener("click", (event) => {
  console.log(event.currentTarget);
});
```

実務では `this` へ依存するより、`event.currentTarget` を明示した方が読みやすく、TypeScriptでも型を扱いやすくなります。

`event.target` は実際にイベントが発生した子要素になる場合があるため、listenerを登録した要素を指す `currentTarget` とは区別します。

## constructor呼び出し

`new` でconstructableな関数を呼ぶと、新しく作られるオブジェクトが `this` として使われます。

```js
function User(name) {
  this.name = name;
}

const user = new User("Taro");
```

この規則は、メソッド呼び出しとは別です。

## thisを使わない方がよい場合

単純な変換処理では、引数と戻り値で表現した方が依存が明確です。

```js
function formatUserName(user) {
  return user.name.trim();
}
```

`this` が適しているのは、オブジェクトの状態と振る舞いをまとめ、そのオブジェクト経由で呼ぶことが自然な場合です。

コールバックへ頻繁に渡す処理、独立した計算、差し替えたい依存まで `this` に入れると、呼び出し条件が見えにくくなります。

## デバッグするときの確認順

`this` が期待と違うときは、次を確認します。

1. アロー関数か通常関数か
2. 実際の呼び出し式にreceiverがあるか
3. メソッドを変数やコールバックへ渡していないか
4. `bind`、`call`、`apply` を使っているか
5. script、module、CommonJSのどの環境か

## まとめ

- 通常関数の `this` は主に呼び出し方で決まる
- `obj.method()` では `obj` が `this`
- メソッドを取り出すとreceiverを失う
- アロー関数は外側の `this` を参照する
- ES Modulesは常にstrict mode
- DOMでは `event.currentTarget` を明示すると分かりやすい

「この関数はどこに書かれたか」ではなく、「どの構文で作られ、どの式で呼ばれたか」を追うことが、`this` を理解する近道です。

## 参考資料

- [MDN: this](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Operators/this)
- [MDN: Function.prototype.bind](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)
- [MDN: Event.currentTarget](https://developer.mozilla.org/docs/Web/API/Event/currentTarget)
- [ECMAScript Language Specification: ResolveThisBinding](https://tc39.es/ecma262/multipage/executable-code-and-execution-contexts.html#sec-resolvethisbinding)
