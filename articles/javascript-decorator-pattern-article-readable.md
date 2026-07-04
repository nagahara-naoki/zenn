# JavaScriptで学ぶDecoratorパターン！RxJS operatorsから見る「元の処理に機能を足す設計」

既存の処理に、あとから共通処理を追加したくなることはありませんか？

例えば、記事を保存する関数があります。

```js
function savePost(post) {
  console.log("記事を保存しました", post);
}
```

ここにログを追加したいとします。

```js
function savePost(post) {
  console.log("保存処理を開始します");

  console.log("記事を保存しました", post);

  console.log("保存処理が完了しました");
}
```

これでも動きます。

でも、同じようなログを他の関数にも追加したくなったらどうでしょうか？

- 保存処理にログを足したい
- API処理にエラーハンドリングを足したい
- 重い処理にキャッシュを足したい
- 認証チェックを足したい
- 実行時間の計測を足したい

このような共通処理を、元の関数に直接書き続けると、コードがどんどん読みにくくなります。

そこで使えるのが **Decoratorパターン** です。

---

## Decoratorパターンとは？

Decoratorパターンは、**元の処理を直接変更せず、外側から機能を追加するパターン**です。

簡単に言うと、

> 元の関数やオブジェクトを包んで、新しい振る舞いを足す

という考え方です。

JavaScriptでは、関数を受け取り、機能を追加した新しい関数を返す形で書くことが多いです。

```js
function withLogging(fn) {
  return function (...args) {
    console.log("処理開始");

    const result = fn(...args);

    console.log("処理終了");

    return result;
  };
}
```

使う側です。

```js
function savePost(post) {
  console.log("記事を保存しました", post);
}

const savePostWithLogging = withLogging(savePost);

savePostWithLogging({
  title: "Decoratorパターン入門",
});
```

元の `savePost` は変更していません。

それでも、ログ機能を追加できています。

---

## decorator構文とは違うの？

ここは少し注意です。

JavaScript / TypeScriptには、`@decorator` のようなdecorator構文があります。

```ts
class Todo {
  @observable accessor title = "";
}
```

これは言語機能としてのdecorator構文です。

一方、この記事で扱うDecoratorパターンは、デザインパターンとしての考え方です。

もちろん関係はありますが、完全に同じものではありません。

この記事では主に、次の考え方を扱います。

```txt
元の処理を直接変えない
  ↓
外側から包む
  ↓
追加の振る舞いを足す
```

---

## Decoratorを使わない場合の問題

保存処理と削除処理に、同じようなログを追加した例です。

```js
function savePost(post) {
  console.log("処理開始");

  console.log("記事を保存しました", post);

  console.log("処理終了");
}

function deletePost(id) {
  console.log("処理開始");

  console.log(`記事 ${id} を削除しました`);

  console.log("処理終了");
}
```

このコードには、次の問題があります。

1. ログ処理が複数箇所に散らばる
2. 本来の処理がログに埋もれる
3. ログの形式を変えるときに修正箇所が増える

本来、`savePost` は保存処理に集中したいはずです。

でも、ログ処理が混ざることで、関数の責務が増えています。

---

## Decorator関数で改善する

ログ処理を、関数を包む形で外に出します。

```js
function withLogging(fn) {
  return function (...args) {
    console.log("処理開始");

    const result = fn(...args);

    console.log("処理終了");

    return result;
  };
}
```

元の関数です。

```js
function savePost(post) {
  console.log("記事を保存しました", post);
}

function deletePost(id) {
  console.log(`記事 ${id} を削除しました`);
}
```

Decoratorを適用します。

```js
const savePostWithLogging = withLogging(savePost);
const deletePostWithLogging = withLogging(deletePost);

savePostWithLogging({
  title: "Decoratorパターン",
});

deletePostWithLogging(1);
```

元の関数を直接変更せずに、ログ機能を追加できました。

---

## Decoratorで何が良くなるのか？

### 1. 元の処理をシンプルに保てる

`savePost` は保存だけに集中できます。

```js
function savePost(post) {
  console.log("記事を保存しました", post);
}
```

ログ処理は `withLogging` に分離されます。

---

### 2. 共通処理を再利用できる

`withLogging` は、他の関数にも使えます。

```js
const createUserWithLogging = withLogging(createUser);
const updateUserWithLogging = withLogging(updateUser);
```

ログの追加方法を1か所にまとめられます。

---

### 3. 必要なときだけ機能を足せる

ログなし版とログあり版を使い分けることもできます。

```js
savePost(post);

savePostWithLogging(post);
```

元の関数を残したまま、追加機能つきの関数を作れるのがDecoratorの強みです。

---

## 非同期処理にも使える

実務では、API通信のような非同期関数を包むことも多いです。

```js
function withErrorHandling(fn) {
  return async function (...args) {
    try {
      return await fn(...args);
    } catch (error) {
      console.error("エラーが発生しました", error);

      throw error;
    }
  };
}
```

使う側です。

```js
async function fetchUser(id) {
  const response = await fetch(`/api/users/${id}`);

  if (!response.ok) {
    throw new Error("ユーザー取得に失敗しました");
  }

  return response.json();
}

const safeFetchUser = withErrorHandling(fetchUser);
```

`fetchUser` の中に毎回 `try/catch` を書かずに、エラーハンドリングを追加できます。

---

## 複数のDecoratorを重ねる

Decoratorは重ねることもできます。

```js
const enhancedFetchUser = withLogging(
  withErrorHandling(fetchUser),
);
```

この場合、`fetchUser` に次の機能を追加しています。

- エラーハンドリング
- ログ出力

ただし、重ねすぎると読みにくくなります。

```js
const handler = withLogging(
  withAuth(
    withCache(
      withErrorHandling(fetchUser),
    ),
  ),
);
```

このようになると、何がどの順番で動くのか追いづらくなります。

Decoratorは便利ですが、使いすぎには注意が必要です。

---

## 有名OSSではどう使われている？

Decoratorパターンに近い考え方は、**RxJSのpipeable operators** に見ることができます。

RxJSでは、Observableに対して `map`、`filter`、`tap` などのoperatorをつなげられます。

```ts
import { of, map, filter, tap } from "rxjs";

of(1, 2, 3, 4)
  .pipe(
    filter((value) => value % 2 === 0),
    map((value) => value * 10),
    tap((value) => console.log("value:", value)),
  )
  .subscribe();
```

このコードでは、元のObservableに対して、外側から処理を重ねています。

- `filter` で値を絞り込む
- `map` で値を変換する
- `tap` でログを出す

元のObservableを直接変更しているわけではありません。

---

## RxJS operatorsをDecoratorとして見る

RxJS公式ガイドでは、pipeable operatorは **Observableを入力として受け取り、別のObservableを出力するpure function** と説明されています。

また、pipeable operatorは既存のObservableインスタンスを変更せず、新しいObservableを返すとも説明されています。

これはDecoratorパターンとかなり相性が良い考え方です。

```txt
元のObservable
  ↓
operatorで包む
  ↓
新しいObservableを返す
```

例えば、`map` は元のObservableを直接変えません。

```ts
const source$ = of(1, 2, 3);

const doubled$ = source$.pipe(
  map((value) => value * 2),
);
```

`source$` はそのままです。

`doubled$` は、値を2倍にする振る舞いが追加された新しいObservableです。

---

## 自作コードでイメージする

かなり単純化して、operatorのイメージを書いてみます。

```js
function map(project) {
  return function (source) {
    return {
      subscribe(observer) {
        return source.subscribe({
          next(value) {
            observer.next(project(value));
          },
        });
      },
    };
  };
}
```

これは本物のRxJS実装ではありません。

ただし、考え方としてはこうです。

```txt
source Observableを受け取る
  ↓
sourceのsubscribeを利用する
  ↓
値に処理を加える
  ↓
新しいObservableとして返す
```

元のObservableを直接変更せず、外側から振る舞いを追加しています。

ここがDecorator的です。

---

## RxJS operatorsから学べること

RxJS operatorsから学べるのは、Decoratorは関数だけでなく、**値の流れにも機能を追加できる** ということです。

例えば、APIレスポンスのObservableがあるとします。

```ts
const user$ = fetchUser$();
```

ここに、変換やログやエラー処理を足せます。

```ts
const safeUser$ = user$.pipe(
  tap(() => console.log("ユーザー取得開始")),
  map((user) => ({
    ...user,
    displayName: user.name.toUpperCase(),
  })),
);
```

元の `user$` を直接変更せず、新しいObservableとして機能を追加しています。

この考え方は、Decoratorパターンにかなり近いです。

---

## 補足：NgRxのmeta-reducersもDecorator的

補足として、NgRxの **meta-reducers** もDecorator的に見ることができます。

NgRxのmeta-reducersは、通常のreducerが呼ばれる前にactionをpre-processできる仕組みです。

通常のreducerは、stateとactionを受け取り、新しいstateを返します。

```ts
function counterReducer(state, action) {
  if (action.type === "increment") {
    return {
      count: state.count + 1,
    };
  }

  return state;
}
```

meta-reducerは、reducerを受け取り、新しいreducerを返します。

```ts
function loggerMetaReducer(reducer) {
  return function (state, action) {
    console.log("action", action);

    const nextState = reducer(state, action);

    console.log("nextState", nextState);

    return nextState;
  };
}
```

これはDecoratorとしてかなりわかりやすいです。

元のreducerを直接変更せず、ログ機能を追加しています。

---

## NgRx meta-reducersをDecoratorとして見る

NgRxでは、meta-reducersを配列で設定できます。

```ts
export const metaReducers = [
  loggerMetaReducer,
];
```

`loggerMetaReducer` は、元のreducerを包みます。

```ts
const enhancedReducer = loggerMetaReducer(counterReducer);
```

この構造は、先ほどの `withLogging` と似ています。

```js
const savePostWithLogging = withLogging(savePost);
```

対応関係はこうです。

| 自作Decorator | NgRx meta-reducer |
|---|---|
| `withLogging(fn)` | `loggerMetaReducer(reducer)` |
| 元の関数 | 元のreducer |
| ログを追加した関数 | ログを追加したreducer |

NgRxのmeta-reducerは、reducerの外側から共通処理を追加する仕組みとして読めます。

---

## 実務でDecoratorを使うならどこ？

Decoratorパターンは、次のような場面で使いやすいです。

- ログ出力を追加したいとき
- エラーハンドリングを共通化したいとき
- 認証チェックを追加したいとき
- キャッシュを追加したいとき
- 実行時間を計測したいとき
- ReactコンポーネントをHOCで包みたいとき
- reducerやObservableに共通処理を足したいとき

例えば、認証チェックを追加するDecoratorです。

```js
function withAuth(fn) {
  return function (...args) {
    if (!isLoggedIn()) {
      throw new Error("ログインしてください");
    }

    return fn(...args);
  };
}
```

使う側です。

```js
const savePostWithAuth = withAuth(savePost);
```

元の `savePost` を直接変更せず、認証チェックを追加できます。

---

## 使いすぎには注意

Decoratorは便利ですが、使いすぎると処理の流れが見えにくくなります。

特に、次のように何重にも包むと読みにくいです。

```js
const handler = withLogging(
  withAuth(
    withCache(
      withErrorHandling(fetchUser),
    ),
  ),
);
```

このコードでは、どの順番で何が実行されるのかを追う必要があります。

また、Decoratorの中で副作用を増やしすぎるのも危険です。

```js
function withAnalytics(fn) {
  return function (...args) {
    sendAnalytics();

    return fn(...args);
  };
}
```

呼び出し側から見ると、関数を呼ぶだけで分析イベントが送られます。

便利ですが、隠れた副作用が増えると予測しづらくなります。

Decoratorは、**共通して追加したい振る舞いがあるとき** に使うのが良いです。

単純な処理まで無理に包む必要はありません。

---

## まとめ

Decoratorパターンは、**元の処理を直接変更せず、外側から機能を追加するパターン**です。

JavaScriptでは、関数を受け取り、機能を追加した新しい関数を返す形で自然に書けます。

RxJSのpipeable operatorsは、Observableを入力として受け取り、新しいObservableを返します。

元のObservableを変更せず、`map`、`filter`、`tap` などの機能を外側から重ねられる点で、Decorator的な考え方として理解できます。

NgRxのmeta-reducersも、reducerを包んでログや前処理などを追加できるため、Decorator的に読むことができます。

Decoratorで大事なのは、**本体処理と追加処理を分けること** です。

既存の関数やObservableやreducerに、ログ・認証・キャッシュ・エラー処理などを足したくなったら、こう考えてみるとよいです。

> この機能、元の処理を書き換えずに外側から足せないかな？

そう思ったときが、Decoratorパターンを使うタイミングです！

---

## 参考リンク

- [RxJS Operators guide](https://rxjs.dev/guide/operators)
- [RxJS map operator source](https://github.com/ReactiveX/rxjs/blob/master/packages/rxjs/src/internal/operators/map.ts)
- [NgRx Meta-Reducers docs](https://ngrx.io/guide/store/metareducers)
