---
title: "JavaScriptのthisを「呼び出し方」から理解する"
emoji: "🤖"
type: "tech"
topics: ["javascript", "this", "初心者"]
published: true
---

オブジェクトのメソッドは動くのに、変数へ取り出した途端 `this` が `undefined` になる。イベントハンドラーをアロー関数へ変えたら参照先が変わる。JavaScriptの `this` は、関数の定義だけを見ても判断できないことが混乱の原因です。

先に結論を述べると、通常の関数の `this` は、主に**関数をどの形で呼び出したか**で決まります。`obj.method()` なら `obj`、`func.call(obj)` なら明示した `obj`、`new Func()` なら新しく作られるオブジェクトです。単独の `func()` はstrict modeで `undefined` になります。一方、アロー関数は自身の `this` を作らず、定義された外側の `this` を使います。

この記事では買い物かごの例を育てながら、メソッド呼び出し、関数の切り離し、`call`・`apply`・`bind`、class、constructor、DOMイベント、デバッグ方法まで整理します。最後には、`this` を使わない方が読みやすい場面も判断できるようにします。

## まず呼び出し方を分類する

通常関数について、最初に確認する表です。アロー関数には別の規則があります。

| 呼び出し方 | 関数内の `this` |
| --- | --- |
| `cart.total()` | `cart` |
| `const f = cart.total; f()` | strict modeでは `undefined` |
| `f.call(cart)` / `f.apply(cart)` | 第1引数の `cart` |
| `const g = f.bind(cart); g()` | bindした `cart` |
| `new Cart()` | 新しく生成されるオブジェクト |
| アロー関数 `() => this` | 定義された外側の `this` |

複数の規則が重なるときは、概ね `new` によるconstructor呼び出し、`bind`・`call`・`apply` による明示指定、`obj.method()` のメソッド呼び出し、単独呼び出しの順に確認します。ただしbound functionを `new` で呼ぶ場合やアロー関数など、単純な順位表だけでは説明できない点も後で扱います。

## メソッド呼び出しでは左側のオブジェクトを見る

買い物かごオブジェクトを作ります。`cart.total()` という呼び出しでは、参照の基点になった `cart` が `this` です。

```js
const cart = {
  taxRate: 0.1,
  items: [
    { name: "本", unitPrice: 1200, quantity: 2 },
    { name: "ペン", unitPrice: 300, quantity: 1 },
  ],

  subtotal() {
    return this.items.reduce(
      (sum, item) => sum + item.unitPrice * item.quantity,
      0,
    );
  },

  total() {
    return Math.floor(this.subtotal() * (1 + this.taxRate));
  },
};

console.log(cart.total()); // 2970
```

`total` を定義した場所が `this` を永久にcartへ結び付けたわけではありません。同じ関数を別のオブジェクトのプロパティへ置き、そのオブジェクト経由で呼べば `this` は変わります。

```js
const taxFreeCart = {
  taxRate: 0,
  items: cart.items,
  subtotal: cart.subtotal,
  total: cart.total,
};

console.log(taxFreeCart.total()); // 2700
```

「ドットの左側」と覚える方法は多くの例で使えますが、正確にはCallExpressionを評価して得たReferenceのbase valueが `this` 値に関わります。普段は、**呼び出しの瞬間に、どのオブジェクトから関数を取得したか**を見ると十分です。

## メソッドを取り出すと関係が外れる

メソッドを変数へ代入すると、変数には関数オブジェクトだけが入ります。「元はcartのメソッドだった」という所有者情報が自動で保存されるわけではありません。

```js
const detachedTotal = cart.total;

detachedTotal();
// ES Modulesやstrict modeでは、thisがundefinedになりエラー
```

分割代入でも同じです。

```js
const { subtotal } = cart;
subtotal(); // this.itemsを読めない
```

メソッドを配列API、Promise、タイマー、イベントAPIなどへそのまま渡すときも、呼び出すのは受け取った側です。元のオブジェクトは保持されません。

```js
const totals = [cart, taxFreeCart].map(cart.total);
// mapはコールバックをcartのメソッドとして呼ばない
```

対策は「呼び出しを包む」「bindする」「そもそも `this` を引数へ変える」の三つです。短期利用のコールバックなら、アロー関数でメソッド呼び出しを包む方法が最も局所的です。

```js
const totals = [cart, taxFreeCart].map(
  (targetCart) => targetCart.total(),
);

setTimeout(() => {
  console.log(cart.total());
}, 0);
```

ここでcartを `this` にしているのはアロー関数ではありません。アロー関数の本体で、改めて `cart.total()` という正しいメソッド呼び出しを行っています。この違いを理解すると、「アロー関数ならthis問題がすべて解決する」という誤解を避けられます。

## 単独呼び出しとstrict mode

通常関数を `func()` と単独で呼ぶと、strict modeでは `this` は `undefined` です。ES Modulesとclass本体は常にstrict modeとして扱われます。

```js
"use strict";

function showThis() {
  console.log(this);
}

showThis(); // undefined
```

非strictの古典的scriptでは、単独呼び出しの `this` がグローバルオブジェクトへ置換されることがあります。しかし、現代のコードでこの挙動へ依存してはいけません。実行形式をscriptからmoduleへ変えるだけで壊れ、ブラウザーとNode.jsの最上位環境も同一ではありません。

グローバルオブジェクトが本当に必要なら `globalThis` を使い、依存であることを明示します。ただし多くの場合、必要な値を引数で受け取る方がテストと再利用が容易です。

```js
function readLocale(environment) {
  return environment.navigator?.language ?? "ja-JP";
}

const locale = readLocale(globalThis);
```

`this` とグローバルスコープも分けて考えます。トップレベルの `this`、トップレベルの変数、`globalThis` のプロパティは、script、module、CommonJSなどで関係が異なります。

## call・applyで呼び出し時に指定する

通常関数の `call` と `apply` は、その一回の呼び出しに使う `this` を明示します。違いは引数の渡し方です。`call` は個別、`apply` は配列風の値で渡します。

```js
function calculateTotal(discount = 0, shipping = 0) {
  const subtotal = this.items.reduce(
    (sum, item) => sum + item.unitPrice * item.quantity,
    0,
  );
  const taxable = Math.max(0, subtotal - discount);
  return Math.floor(taxable * (1 + this.taxRate)) + shipping;
}

console.log(calculateTotal.call(cart, 500, 600));
console.log(calculateTotal.apply(cart, [500, 600]));
```

別オブジェクトの処理を借りる用途にも使えますが、借りる側が必要なプロパティを暗黙に満たす必要があります。構造が変わると実行時まで失敗しません。共通計算として再利用したいだけなら、オブジェクトを通常の引数にした関数の方が契約を読みやすくできます。

```js
function calculateCartTotal(targetCart, discount = 0) {
  const subtotal = targetCart.items.reduce(
    (sum, item) => sum + item.unitPrice * item.quantity,
    0,
  );
  return Math.floor(
    Math.max(0, subtotal - discount) * (1 + targetCart.taxRate),
  );
}
```

`call` や `apply` を使っても、アロー関数の `this` は変更できません。引数自体は渡せますが、外側の `this` を参照し続けます。

## bindはthisを固定した新しい関数を返す

`bind` は元の関数を実行せず、指定した `this` と、必要なら先頭の引数を固定した新しいbound functionを返します。

```js
const boundTotal = cart.total.bind(cart);

console.log(boundTotal());
setTimeout(boundTotal, 0);
```

`bind` の戻り値は元の関数とは別オブジェクトです。イベントを解除する場合、登録時と同じbound functionを保存して使う必要があります。

```js
const view = {
  cart,
  render() {
    output.textContent = String(this.cart.total());
  },
};

const handleClick = view.render.bind(view);
button.addEventListener("click", handleClick);

// 破棄時
button.removeEventListener("click", handleClick);
```

次の書き方では、`bind` を呼ぶたびに新しい関数ができるため解除できません。

```js
button.addEventListener("click", view.render.bind(view));
button.removeEventListener("click", view.render.bind(view)); // 別の関数
```

何度も `bind` を重ねても、最初にbindした `this` は後のbindで置き換わりません。引数は追加で前置できます。意図が分かりづらくなるため、通常は一度だけbindします。

## アロー関数は外側のthisを使う

アロー関数は、呼び出し方による `this` bindingを作りません。定義されたレキシカルな外側の `this` を参照します。そのため、メソッド内のコールバックで同じインスタンスを使うときに便利です。

```js
const cartView = {
  cart,

  renderItems() {
    return this.cart.items.map((item) => {
      return `${item.name}: ${item.quantity}個`;
    });
  },
};
```

`map` がアロー関数をどのように呼んでも、アロー関数は `renderItems` の `this` を使います。以前は `const self = this` と保存したり、第2引数の `thisArg` を渡したりしましたが、レキシカルに使いたい場面ではアロー関数が自然です。

反対に、オブジェクトのメソッド自体をアロー関数で定義しても、呼び出したオブジェクトは受け取りません。

```js
const badCart = {
  items: [],
  total: () => {
    return this.items.length; // badCartのthisではない
  },
};
```

`call`、`apply`、`bind` でもアロー関数の `this` は変えられません。「後で呼び出し元を差し替える関数」には通常関数、「外側の文脈を維持する短いコールバック」にはアロー関数、と使い分けます。

## classのメソッドは自動でbindされない

class構文でも、prototype methodの `this` は呼び出し方で決まります。classに書いたからインスタンスへ固定されるわけではありません。

```js
class Cart {
  constructor(items, taxRate = 0.1) {
    this.items = items;
    this.taxRate = taxRate;
  }

  subtotal() {
    return this.items.reduce(
      (sum, item) => sum + item.unitPrice * item.quantity,
      0,
    );
  }

  total() {
    return Math.floor(this.subtotal() * (1 + this.taxRate));
  }
}

const instance = new Cart(cart.items);
const detached = instance.total;
// detached(); // thisがundefined
```

UIコールバックとして頻繁に渡すメソッドは、constructorで一度bindし、解除にも使える参照を保持できます。

```js
class CartView {
  constructor(cart, button) {
    this.cart = cart;
    this.button = button;
    this.handleClick = this.handleClick.bind(this);
  }

  mount() {
    this.button.addEventListener("click", this.handleClick);
  }

  unmount() {
    this.button.removeEventListener("click", this.handleClick);
  }

  handleClick() {
    console.log(this.cart.total());
  }
}
```

もう一つの方法は、public fieldへアロー関数を代入することです。この書き方はインスタンス初期化時の `this` を閉じ込めます。ただしprototypeで一つのメソッドを共有する方式と違い、各インスタンスに関数が作られます。必要なコールバックだけに限定すると、意図とコストのバランスを取りやすくなります。

```js
class CartView {
  constructor(cart) {
    this.cart = cart;
  }

  handleClick = () => {
    console.log(this.cart.total());
  };
}
```

## newによるconstructor呼び出し

constructableな通常関数やclassを `new` で呼ぶと、新しいオブジェクトが作られ、constructor本体の `this` になります。明示的に別のオブジェクトをreturnしない限り、その新しいオブジェクトが結果です。

```js
function Cart(items, taxRate = 0.1) {
  this.items = items;
  this.taxRate = taxRate;
}

Cart.prototype.total = function total() {
  const subtotal = this.items.reduce(
    (sum, item) => sum + item.unitPrice * item.quantity,
    0,
  );
  return Math.floor(subtotal * (1 + this.taxRate));
};

const created = new Cart(cart.items);
console.log(created.total());
```

`new` で呼べないのは、アロー関数が `[[Construct]]` を持たないためです。classは `new` なしで呼ぶと例外になり、意図しないグローバル書き込みを防ぎます。インスタンス生成を表すなら、古いconstructor関数よりclassか、`createCart()` のようなファクトリー関数が明確です。

bound functionを `new` で呼ぶと、bind時に指定した `this` はconstructor用の新しいオブジェクトへ置き換わります。ただし固定した引数は使われます。この例外があるため、「bindが常にthisを変更不能に固定する」と断言するのは不正確です。

## DOMイベントではcurrentTargetを優先する

`addEventListener` へ通常関数を渡した場合、DOM仕様ではコールバック呼び出し時の `this` はイベントの `currentTarget` になります。アロー関数を渡すと、外側の `this` のままです。

```js
button.addEventListener("click", function (event) {
  console.log(this === event.currentTarget); // true
  this.disabled = true;
});

button.addEventListener("click", (event) => {
  event.currentTarget.disabled = true;
});
```

DOMコードでは `this` より `event.currentTarget` を使うと意図が明示的です。`event.target` は実際にイベントが発生した子要素になり得るので、リスナーを登録した要素とは限りません。

イベントAPIによってコールバックの呼び出し規則は異なります。配列メソッドには任意の `thisArg` を受け取るものもありますが、Promiseのハンドラーに元オブジェクトを補う規則はありません。「コールバックならthisはこれ」と一般化せず、受け取るAPIの契約を確認します。

## thisを使うべき場面と使わない場面

`this` は、同じ操作を複数のオブジェクトへ適用し、呼び出し元をreceiverとして扱う設計に向いています。classのインスタンスメソッド、オブジェクトの振る舞い、DOMの一部のコールバック契約が代表例です。

一方、単純な計算、データ変換、依存が少ない処理では、必要な値を引数へ明示した方が理解しやすくなります。関数を見ただけで入力が分かり、切り離しても壊れず、テストでbindする必要もありません。

```js
function calculateTotal({ items, taxRate }, discount = 0) {
  const subtotal = items.reduce(
    (sum, item) => sum + item.unitPrice * item.quantity,
    0,
  );
  return Math.floor(Math.max(0, subtotal - discount) * (1 + taxRate));
}

const total = calculateTotal(cart, 500);
```

可変状態を隠したいだけなら、クロージャやモジュールスコープも候補です。すべてをclassと `this` で表す必要はありません。反対に、引数の先頭へ同じcontextオブジェクトを何層も渡すなら、状態と操作をオブジェクトへまとめる方が自然かもしれません。

判断基準は「`this` を使えるか」ではなく、「呼び出したreceiverによって振る舞いが決まることが、利用側へとって自然か」です。

## デバッグするときの確認順

`Cannot read properties of undefined` などがメソッド内で起きたら、値を直す前に呼び出し方を確認します。

1. 対象が通常関数かアロー関数かを確認する。
2. 実際の呼び出し式全体を見て、`obj.method()` か `func()` かを確認する。
3. 分割代入、引数渡し、コールバック登録でメソッドを切り離していないか確認する。
4. `call`、`apply`、`bind`、`new` が使われていないか確認する。
5. class fieldのアロー関数なら、インスタンスごとに作られた関数か確認する。
6. DOMイベントなら `target` と `currentTarget` を区別する。
7. ES Module、strict mode、テスト環境など実行形式の違いを確認する。

ブレークポイントで停止し、consoleで `this`、対象関数、コールスタックを確認します。メソッド内部へログを追加するだけでは、誰がどの式で呼んだかを見落とすことがあります。呼び出し元へブレークポイントを置くか、スタックを上へたどります。

テストでは、公開されている想定の呼び出し方で振る舞いを確認します。同じメソッドを別receiverへ借用できることが契約でないなら、その柔軟性までテストする必要はありません。逆にコールバックとして渡すAPIなら、切り離された状態でも動くようbind済みか、ラッパーを要求するのかをテスト名で明示します。

## よくある誤解

「`this` は関数を定義したオブジェクト」は誤りです。通常関数は呼び出し時に決まり、同じ関数でもreceiverを変えられます。「アロー関数は自身を `this` にする」も誤りで、外側の `this` を使います。

「bindすれば元の関数が変わる」も誤りです。bindは新しい関数を返すので、戻り値を保存しなければ効果を利用できません。「classのメソッドは自動bind」でもありません。prototype methodを取り出せば `this` は外れます。

また、DevTools consoleで式を試した結果を、そのままES Moduleや本番コードへ一般化しないことも大切です。consoleの評価コンテキスト、ページのscript種別、Node.jsのモジュール方式で最上位の `this` は異なります。問題が起きたコードと同じ実行形式で最小再現を作ります。

## まとめ

JavaScriptの `this` は、通常関数なら定義場所より呼び出し方を先に見ます。メソッドを変数やコールバックへ渡すと、元オブジェクトとの関係は自動で保存されません。例外はアロー関数です。自身のbindingを作らず、外側の `this` を使います。

- `obj.method()` では、呼び出しのreceiverである `obj` が `this` になる。
- メソッドを取り出した `func()` は、strict modeで `this` が `undefined` になる。
- 一回だけ指定するなら `call` / `apply`、新しい固定関数が必要なら `bind` を使う。
- アロー関数は外側の `this` を保つが、オブジェクトのメソッド代わりにはならない。
- classのprototype methodも自動bindされない。登録と解除には同じ関数参照を使う。
- `new` は新しいオブジェクトを `this` にし、アロー関数はconstructorにできない。
- 単純な計算では、`this` より明示引数の方が読みやすいことが多い。

迷ったときは「この関数を誰のプロパティとして取得し、どの構文で呼んだか」を一行で書き出してください。それでもreceiverという考え方が不自然なら、`this` を使わず必要な値を引数へ渡す設計が、より素直な可能性があります。

## 参考資料

- [ECMAScript: Ordinary Function Calls](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-ordinary-function-calls)
- [ECMAScript: EvaluateCall](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-evaluatecall)
- [ECMAScript: Function.prototype.call](https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-function.prototype.call)
- [ECMAScript: Bound Function Exotic Objects](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-bound-function-exotic-objects)
- [ECMAScript: Arrow Function Definitions](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-arrow-function-definitions)
- [DOM Standard: Event listener callback](https://dom.spec.whatwg.org/#concept-event-listener-inner-invoke)
- [MDN: this](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Operators/this)
