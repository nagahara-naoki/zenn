---
title: "Typed FormsとSignal Forms"
---

前章で、Template-drivenとReactiveという2つのフォーム方式を学びました。この章では、その先の進化を扱います。ひとつは、Reactive Formsを型安全にしたTyped Forms。もうひとつは、Angular 22で安定版となった、まったく新しいSignal Formsです。

フォームは、Angularが継続的に改善を重ねてきた領域です。Typed Formsは型の穴を塞ぎ、Signal Formsは、これまで学んだSignalの考え方をフォームに持ち込みました。とくにSignal Formsは、モダンAngularのフォームの中心となることが期待される、重要な新機能です。この章では、両者の考え方と書き方を、これまでの方式と比較しながら学びます。

:::message
**この章で学ぶこと**

- Typed Formsによる型安全なReactive Forms
- Signal Formsの考え方
- `form()`とスキーマによる検証
- 3つのフォーム方式の位置づけ
:::

## Typed Forms

Reactive Formsは、長らく型の面に弱点を抱えていました。フォームから取り出した値の型が、必ずしも正確でなく、思わぬ実行時エラーを招くことがあったのです。この問題を解決したのが、Angular 14（2022年）で導入されたTyped Formsです。

Typed Formsでは、フォームの構造から、値の型が正確に導かれます。前章の`FormBuilder`による定義は、実はすでにTyped Formsとして動いています。

```ts:src/app/profile-form.ts
protected readonly form = this.fb.group({
  name: ['', Validators.required],
  age: [0],
});

save(): void {
  const value = this.form.value;
  // value.name は string | undefined、value.age は number | undefined と型付く
}
```

`form.value`の型が、定義した構造（`name`は文字列、`age`は数値）から自動で導かれます。存在しない項目にアクセスしようとすれば、コンパイル時にエラーになります。第5章で学んだTypeScriptの型の恩恵が、フォームにも及ぶわけです。現在のReactive Formsは、標準でこのTyped Formsとして動くため、特別な設定は要りません。型の安全性が、最初から得られます。

## Signal Formsの登場

Reactive Formsは堅牢ですが、RxJSベースであり、値の変化は`valueChanges`というObservableで受け取ります。第6部・第8部で見たように、モダンAngularの状態管理はSignalへ移りつつあります。フォームだけがObservableのままでは、アプリ全体の一貫性が損なわれます。

そこで登場したのが、Signal Formsです。Angular 22（2026年）で安定版になった、Signalを土台とする新しいフォームの仕組みです。フォームの状態がSignalとして表現されるため、これまで学んできたSignalの考え方が、そのままフォームに通用します。`@angular/forms/signals`から機能をimportして使います。

Signal Formsの核心は、フォームのデータを、ふつうのSignalとして持つことです。そのSignalモデルから、`form()`関数でフォームを組み立てます。

## form()でフォームを作る

Signal Formsでは、まずフォームのデータをSignalで定義します。次に、それを`form()`に渡し、第2引数のスキーマ関数で検証ルールを宣言します。

```ts:src/app/login-form.ts
import { Component, signal } from '@angular/core';
import { form, required, email } from '@angular/forms/signals';

@Component({ selector: 'app-login-form', template: `...` })
export class LoginForm {
  // フォームのデータを、ふつうのSignalで持つ
  protected readonly loginModel = signal({ email: '', password: '' });

  // モデルとスキーマから、フォームを組み立てる
  protected readonly loginForm = form(this.loginModel, (path) => {
    required(path.email, { message: 'メールアドレスは必須です' });
    email(path.email, { message: '正しい形式で入力してください' });
    required(path.password, { message: 'パスワードは必須です' });
  });
}
```

`form(this.loginModel, ...)`の第1引数が、データを持つSignalです。第2引数のスキーマ関数は、`path`を受け取り、各項目に検証ルールを適用します。`required(path.email)`のように、どの項目にどの検証を課すかを、宣言的に書きます。検証ルール（`required`・`email`など）は、組み合わせられる関数として提供されます。

## テンプレートとの結びつけと状態

作ったフォームは、`[formField]`ディレクティブで入力欄に結びつけます。

```html
<form>
  <input type="email" [formField]="loginForm.email" />
  @if (loginForm.email().invalid() && loginForm.email().touched()) {
    <p>{{ loginForm.email().errors() }}</p>
  }

  <input type="password" [formField]="loginForm.password" />

  <button [disabled]="loginForm().invalid()">ログイン</button>
</form>
```

`[formField]="loginForm.email"`で、フォームの`email`項目と入力欄が結びつきます。双方向の同期は、Signal Formsが自動で行います。各項目の状態は、その項目を関数として呼び出して読み取ります。`loginForm.email()`で項目の状態にアクセスし、さらに`.value()`で現在値、`.invalid()`で検証結果、`.errors()`でエラー、`.touched()`で操作済みかを、いずれもSignalとして取得できます。

すべてがSignalなので、テンプレートでの表示も、これまでのSignalとまったく同じ感覚で書けます。`valueChanges`の購読も、その解除も要りません。フォームの状態が、アプリのほかの状態と同じ土俵に乗るのです。これが、Signal Formsの最大の利点です。

## 3つの方式の位置づけ

これで、フォームの方式が3つになりました。それぞれの位置づけを整理します。

| 方式 | 中心 | 特徴 |
|---|---|---|
| Template-driven | テンプレート | 手軽。単純なフォーム向け |
| Reactive | クラス（RxJSベース） | 堅牢。従来の複雑なフォームの定番 |
| Signal Forms | Signalモデル | 型安全でSignal一貫。v22の新標準候補 |

Signal Formsは、Reactive Formsの堅牢さと型安全性を保ちながら、Signalベースである点で、モダンAngularとの一貫性に優れます。Angularは、既存のReactive Formsからの段階的な移行も見据えており、両者を併存させながら移していけるよう配慮されています。

本書では、新規に複雑なフォームを作るなら、Signalベースで書けるSignal Formsを第一の選択肢として推奨します。ただし、Signal Formsは登場して間もないため、既存プロジェクトの多くはReactive Formsで書かれています。当面は、両方を理解しておくことが実務上は重要です。単純なフォームであれば、Template-drivenも引き続き有効です。

## なぜSignal Formsが重要なのか

Signal Formsが重要なのは、フォームの状態を、アプリケーションのほかの状態と同じ「Signal」という土俵に載せる点にあります。第6部で見たように、モダンAngularは状態管理をSignalに統一する方向へ進んでいます。ところが、フォームだけがRxJSベースのままだと、フォームの値をほかのSignalと組み合わせるたびに、変換（`toSignal()`など）が必要でした。

Signal Formsでは、フォームの値も検証状態も、最初からSignalです。`computed()`でフォームの値から別の値を導いたり、`effect()`でフォームの変化に反応したりが、変換なしに、そのまま書けます。フォームが、アプリの状態管理に自然に溶け込むのです。これは、単なる書き方の違いではなく、アプリケーション全体の一貫性という、より大きな利点につながります。第6部から一貫して見てきた「状態はSignalで持つ」という流れの、フォームにおける到達点が、Signal Formsだといえます。

## よくあるつまずき

- **フォームの状態の読み方を間違える**: Signal Formsでは、項目を関数として呼んでから状態を読みます。`loginForm.email().value()`のように、二段階になる点に注意します。
- **方式を混同する**: Template-driven（`ngModel`）、Reactive（`formControlName`）、Signal Forms（`[formField]`）は、それぞれ結びつけ方が違います。1つのフォームでは、方式を統一します。
- **新しさだけで飛びつく**: Signal Formsは有望ですが、既存プロジェクトがReactiveで統一されているなら、無理に混在させず、方針を決めて選びます。
- **検証を検証関数の外に書く**: Signal Formsの検証は、スキーマ関数の中で`required(path.email)`のように宣言します。テンプレート側で条件分岐を駆使して検証を模倣すると、Signal Formsの利点が薄れます。ルールはスキーマに集約します。

## まとめ

- Typed Formsは、Reactive Formsの値に正確な型を与える仕組みで、v14以降は標準です
- Signal Formsは、v22で安定化した、Signalを土台とする新しいフォーム方式です
- `form(model, スキーマ関数)`でフォームを作り、`[formField]`で入力欄に結びつけます
- 状態は`field().value()`・`.invalid()`・`.errors()`のように、Signalとして読み取ります
- **新規の複雑なフォームにはSignal Formsを推奨します。既存のReactive Formsも当面は理解が必要です**

次章では、サーバーとデータをやり取りするHTTP通信の基本を学びます。
