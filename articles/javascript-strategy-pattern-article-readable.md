# JavaScriptで学ぶStrategyパターン！NgRxのcreateReducerから見る「処理を切り替える設計」

条件分岐がどんどん増えて、関数が読みにくくなったことはありませんか？

例えば、会員ランクごとに料金を計算する処理です。

```js
function calculatePrice(memberType, price) {
  if (memberType === "normal") {
    return price;
  }

  if (memberType === "silver") {
    return price * 0.9;
  }

  if (memberType === "gold") {
    return price * 0.8;
  }

  if (memberType === "vip") {
    return price * 0.7;
  }

  return price;
}
```

最初はこれで十分です。

でも、会員ランクが増えたり、計算ルールが複雑になったりすると、`if` や `switch` がどんどん大きくなります。

- 条件が増える
- 1つの関数が肥大化する
- 新しい処理を追加するたびに既存コードを触る
- 処理ごとのテストがしづらい
- どの条件で何が動くのか見通しが悪くなる

こういうときに使えるのが **Strategyパターン** です。

---

## Strategyパターンとは？

Strategyパターンは、**処理のやり方を部品として分け、必要に応じて選べるようにするパターン**です。

簡単に言うと、

> 条件分岐の中に処理を詰め込まず、処理そのものを外に出して切り替える

という考え方です。

料金計算であれば、会員ランクごとの計算処理をそれぞれ関数として分けます。

```js
const priceStrategies = {
  normal(price) {
    return price;
  },

  silver(price) {
    return price * 0.9;
  },

  gold(price) {
    return price * 0.8;
  },

  vip(price) {
    return price * 0.7;
  },
};
```

そして、必要な処理を選んで実行します。

```js
function calculatePrice(memberType, price) {
  const strategy = priceStrategies[memberType] || priceStrategies.normal;

  return strategy(price);
}
```

これがStrategyパターンの基本です。

---

## Strategyを使わない場合の問題

もう一度、`if` で書いた料金計算を見てみます。

```js
function calculatePrice(memberType, price) {
  if (memberType === "normal") {
    return price;
  }

  if (memberType === "silver") {
    return price * 0.9;
  }

  if (memberType === "gold") {
    return price * 0.8;
  }

  if (memberType === "vip") {
    return price * 0.7;
  }

  return price;
}
```

このコードには、次の問題があります。

1. 会員ランクが増えるたびに関数が大きくなる
2. 料金計算ルールが1つの関数に集まりすぎる
3. 特定ランクだけを個別にテストしづらい

たとえば、`student` や `campaign` を追加したくなったら、この関数をまた修正する必要があります。

```js
if (memberType === "student") {
  return price * 0.85;
}

if (memberType === "campaign") {
  return price * 0.5;
}
```

これが続くと、`calculatePrice` はどんどん読みにくくなります。

---

## Strategyパターンで改善する

会員ランクごとの計算処理を、関数として分けます。

```js
const priceStrategies = {
  normal(price) {
    return price;
  },

  silver(price) {
    return price * 0.9;
  },

  gold(price) {
    return price * 0.8;
  },

  vip(price) {
    return price * 0.7;
  },
};

function calculatePrice(memberType, price) {
  const strategy = priceStrategies[memberType];

  if (!strategy) {
    throw new Error(`Unknown member type: ${memberType}`);
  }

  return strategy(price);
}
```

使う側です。

```js
const price = calculatePrice("gold", 1000);

console.log(price); // 800
```

`calculatePrice` は、どの処理を使うかを選ぶだけになりました。

実際の計算ルールは、`priceStrategies` に分かれています。

---

## Strategyで何が良くなるのか？

### 1. 処理ごとに分けて管理できる

ゴールド会員の計算はここだけです。

```js
gold(price) {
  return price * 0.8;
}
```

VIP会員の計算はここだけです。

```js
vip(price) {
  return price * 0.7;
}
```

1つの関数にすべて詰め込むより、見通しが良くなります。

---

### 2. 新しい処理を追加しやすい

プラチナ会員を追加したい場合は、strategyを1つ追加します。

```js
const priceStrategies = {
  normal(price) {
    return price;
  },

  gold(price) {
    return price * 0.8;
  },

  platinum(price) {
    return price * 0.6;
  },
};
```

`calculatePrice` の基本構造は変えなくて済みます。

---

### 3. 処理ごとにテストしやすい

strategyが関数として分かれているので、個別にテストできます。

```js
console.log(priceStrategies.gold(1000)); // 800
console.log(priceStrategies.vip(1000)); // 700
```

「goldの料金計算だけを確認する」ということがやりやすくなります。

---

## Strategyはif文を消すためだけのものではない

ここは大事です。

Strategyパターンは、単に `if` や `switch` を消すためのテクニックではありません。

本質は、**変わりやすい処理を意味のある単位で分けること** です。

例えば、次のような単純な条件分岐なら、Strategyにしなくてもよいです。

```js
function getStatusLabel(isActive) {
  return isActive ? "有効" : "無効";
}
```

この程度なら、そのまま書いた方が読みやすいです。

Strategyが役立つのは、次のような場合です。

- 種類ごとに処理内容が違う
- 条件が今後増えそう
- 処理ごとにテストしたい
- 実行する処理を外から差し替えたい
- `if` / `switch` が大きくなりすぎている

---

## 有名OSSではどう使われている？

Strategyパターンに近い考え方は、Angular向け状態管理ライブラリの **NgRx** に見ることができます。

NgRxでは、`createReducer` と `on` を使って、actionごとのstate更新処理を登録します。

```ts
import { createReducer, on } from "@ngrx/store";
import { increment, decrement } from "./counter.actions";

const initialState = {
  count: 0,
};

export const counterReducer = createReducer(
  initialState,

  on(increment, (state) => ({
    ...state,
    count: state.count + 1,
  })),

  on(decrement, (state) => ({
    ...state,
    count: state.count - 1,
  })),
);
```

このコードでは、actionごとに処理が分かれています。

- `increment` のときは count を増やす
- `decrement` のときは count を減らす

つまり、action typeに応じて実行する処理を切り替えています。

---

## NgRxのcreateReducerをStrategyとして見る

NgRxの `createReducer` の実装では、action typeとreducer関数を `Map` に登録しています。

実装をかなり簡略化すると、次のようなイメージです。

```ts
function createReducer(initialState, ...ons) {
  const map = new Map();

  for (const on of ons) {
    for (const type of on.types) {
      map.set(type, on.reducer);
    }
  }

  return function reducer(state = initialState, action) {
    const reducer = map.get(action.type);

    return reducer ? reducer(state, action) : state;
  };
}
```

これはStrategyパターンとして見ると、とてもわかりやすいです。

自作例では、会員種別から料金計算関数を選びました。

```js
const strategy = priceStrategies[memberType];
```

NgRxでは、action typeからreducer関数を選んでいます。

```ts
const reducer = map.get(action.type);
```

対応関係はこうです。

| 自作Strategy | NgRx |
| --- | --- |
| `memberType` | `action.type` |
| `priceStrategies` | `Map<actionType, reducer>` |
| `strategy(price)` | `reducer(state, action)` |

NgRxの `createReducer` は、公式に「Strategyパターンです」と説明されているわけではありません。

ただ、設計の見方としては、**action typeに応じて処理戦略を選ぶ仕組み** と読むことができます。

---

## switch文との違い

従来のReduxや状態管理では、reducerを `switch` で書くことがあります。

```ts
function reducer(state = initialState, action) {
  switch (action.type) {
    case "increment":
      return {
        ...state,
        count: state.count + 1,
      };

    case "decrement":
      return {
        ...state,
        count: state.count - 1,
      };

    default:
      return state;
  }
}
```

これはこれで動きます。

ただし、actionが増えると `switch` が大きくなります。

NgRxの `createReducer` では、actionごとの処理を `on` で分けて登録できます。

```ts
createReducer(
  initialState,
  on(increment, incrementReducer),
  on(decrement, decrementReducer),
);
```

処理を登録し、実行時にaction typeで選ぶ。

この構造がStrategy的です。

---

## NgRxから学べること

NgRxの `createReducer` から学べるのは、Strategyパターンはアプリコードだけでなく、ライブラリ設計にも自然に出てくるということです。

利用者は、こう書きます。

```ts
on(increment, (state) => ({
  ...state,
  count: state.count + 1,
}))
```

ライブラリ内部では、action typeとreducer関数の対応表を作ります。

そして、actionがdispatchされたときに該当するreducerを選んで実行します。

つまり、NgRxは「action typeに応じた処理の切り替え」を、ライブラリAPIとして整理しているわけです。

---

## 補足：Redux ToolkitのcreateReducerもStrategy的

Redux Toolkitの `createReducer` も、Strategy的な見方ができます。

Redux Toolkitでは、builderを使ってactionごとの処理を登録します。

```ts
import { createAction, createReducer } from "@reduxjs/toolkit";

const increment = createAction<number>("increment");
const decrement = createAction<number>("decrement");

const reducer = createReducer(
  {
    count: 0,
  },
  (builder) => {
    builder
      .addCase(increment, (state, action) => {
        state.count += action.payload;
      })
      .addCase(decrement, (state, action) => {
        state.count -= action.payload;
      });
  },
);
```

Redux Toolkit公式ドキュメントでは、`builder.addCase` は特定のaction typeを扱うcase reducerを追加するAPIとして説明されています。

さらに `addMatcher` を使うと、action typeの完全一致だけでなく、条件に合うactionをまとめて処理できます。

```ts
builder.addMatcher(
  (action) => action.type.endsWith("/pending"),
  (state) => {
    state.loading = true;
  },
);
```

これは、処理の選び方をより柔軟にしたStrategy的な設計として見ることができます。

---

## 実務でStrategyを使うならどこ？

Strategyパターンは、次のような場面で使いやすいです。

- 会員種別ごとの料金計算
- 通知方法の切り替え
- バリデーションルールの切り替え
- ソート方法の切り替え
- 決済方法ごとの処理
- action typeごとのstate更新
- 環境ごとの処理切り替え

例えば、通知方法の切り替えです。

```js
const notificationStrategies = {
  email(message) {
    console.log(`メール送信: ${message}`);
  },

  slack(message) {
    console.log(`Slack送信: ${message}`);
  },

  sms(message) {
    console.log(`SMS送信: ${message}`);
  },
};

function sendNotification(type, message) {
  const strategy = notificationStrategies[type];

  if (!strategy) {
    throw new Error(`Unknown notification type: ${type}`);
  }

  strategy(message);
}
```

通知方法が増えたら、strategyを追加します。

```js
notificationStrategies.line = function (message) {
  console.log(`LINE送信: ${message}`);
};
```

大きな `if` 文に処理を追加し続けるより、見通しが良くなります。

---

## 使いすぎには注意

Strategyパターンは便利ですが、使いすぎると逆に読みにくくなります。

例えば、次のような単純な条件分岐なら、Strategyにする必要はあまりありません。

```js
function getRoleLabel(role) {
  if (role === "admin") {
    return "管理者";
  }

  return "一般ユーザー";
}
```

これを無理にStrategyにすると、かえって大げさです。

```js
const roleLabelStrategies = {
  admin() {
    return "管理者";
  },

  user() {
    return "一般ユーザー";
  },
};
```

Strategyが役立つのは、**処理内容がそれなりに大きい場合** です。

単なるラベル出し分けや、今後増えない小さな条件分岐なら、普通に `if` で書いた方が読みやすいです。

---

## まとめ

Strategyパターンは、**処理のやり方を部品として分け、必要に応じて選べるようにするパターン**です。

`if` や `switch` が増えてきたときに、処理を意味のある単位で分けられます。

NgRxの `createReducer` は、action typeとreducer関数を対応づけ、実行時に `action.type` から処理を選ぶ構造を持っています。

Redux Toolkitの `createReducer` も、`builder.addCase` や `addMatcher` によって、actionに応じた処理を登録できます。

Strategyパターンで大事なのは、ただ条件分岐を消すことではありません。

大事なのは、**変わりやすい処理を分けて、選べる形にすること** です。

条件分岐の中に処理が増えてきたら、こう考えてみるとよいです。

> この処理、種類ごとに分けて選べるようにした方がよくない？

そう思ったときが、Strategyパターンを使うタイミングです！

---

## 参考リンク

- [NgRx reducer_creator.ts](https://github.com/ngrx/platform/blob/main/modules/store/src/reducer_creator.ts)
- [NgRx createReducer API](https://ngrx.io/api/store/createReducer)
- [Redux Toolkit createReducer docs](https://redux-toolkit.js.org/api/createreducer)
- [Redux Toolkit createReducer source](https://github.com/reduxjs/redux-toolkit/blob/master/packages/toolkit/src/createReducer.ts)
