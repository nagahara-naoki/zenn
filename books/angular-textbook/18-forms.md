---
title: "Forms（Template-driven・Reactive・Signal Forms）"
---

この章では、利用者からの入力を扱うFormsを学びます。伝統的なTemplate-driven・Reactiveの2方式に加え、v22で安定化したSignal Formsを扱います。

:::message
**この章で学ぶこと**

- Angularにおける2つのフォーム方式
- Template-driven Formsの書き方
- Typed Formsによる型安全なReactive Forms
- Signal Formsの考え方
:::

## Template-driven FormsとReactive Forms

フォームは、利用者からの入力を受け取る、アプリケーションの重要な入り口です。名前の入力、ログイン、検索、設定の変更。これらはすべてフォームです。Angularは、フォームを扱うための仕組みを複数用意しています。この節では、伝統的な2つの方式、Template-driven FormsとReactive Formsを学びます。

この2つは、長らくAngularのフォームの二本柱でした。Template-drivenは手軽で、Reactiveは堅牢です。それぞれに向き不向きがあり、どちらを使うかは、フォームの複雑さによって判断します。次章では、これらに加えてSignal Formsという新しい方式も学びますが、まずは既存の2方式を理解することが、フォーム全体を捉える土台になります。

### 2つのフォーム方式

Angularには、伝統的に2つのフォーム方式があります。

- **Template-driven Forms**: テンプレート側を主役に、フォームを組み立てる方式です。`ngModel`を使い、テンプレートに書いた指定から、Angularがフォームの構造を読み取ります。手軽さが特徴です。
- **Reactive Forms**: クラス側を主役に、フォームを組み立てる方式です。`FormControl`や`FormGroup`をクラスで定義し、それをテンプレートに結びつけます。構造が明示的で、堅牢さが特徴です。

大きな違いは、「フォームの定義が、テンプレートとクラスのどちらにあるか」です。Template-drivenはテンプレートに、Reactiveはクラスに、フォームの中心があります。順に見ていきましょう。

### Template-driven Forms

Template-driven Formsは、手軽にフォームを作れる方式です。使うには、`FormsModule`をimportします。中心となるのが、`ngModel`です。第11章で双方向バインディングに触れましたが、`ngModel`はその代表例で、入力欄とデータを双方向に結びつけます。

```ts:src/app/login-form.ts
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-login-form',
  imports: [FormsModule],
  template: `
    <form (ngSubmit)="submit()">
      <input name="email" [(ngModel)]="email" required />
      <input name="password" type="password" [(ngModel)]="password" required />
      <button type="submit">ログイン</button>
    </form>
  `,
})
export class LoginForm {
  protected email = '';
  protected password = '';

  protected submit(): void {
    console.log(this.email, this.password);
  }
}
```

`[(ngModel)]="email"`で、入力欄と`email`プロパティが双方向に結びつきます。利用者が入力すれば`email`が変わり、`email`を変えれば入力欄も変わります。`required`のような検証も、HTMLの属性のように書けます。`(ngSubmit)`は、フォームが送信されたときに呼ばれるイベントです。

Template-drivenは、直感的で、少ないコードで書けます。単純なフォーム、たとえば項目が少ない設定画面やログインフォームには十分です。一方、フォームが複雑になると、ロジックがテンプレートに散らばり、扱いにくくなる面があります。

### Reactive Forms

Reactive Formsは、フォームの構造をクラス側で明示的に定義する方式です。使うには、`ReactiveFormsModule`をimportします。中心となるのが、`FormControl`（1つの入力）と`FormGroup`（入力のまとまり）です。

```ts:src/app/login-form.ts
import { Component, inject } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

@Component({
  selector: 'app-login-form',
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="email" />
      <input formControlName="password" type="password" />
      <button type="submit" [disabled]="form.invalid">ログイン</button>
    </form>
  `,
})
export class LoginForm {
  private readonly fb = inject(FormBuilder);

  protected readonly form = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', Validators.required],
  });

  protected submit(): void {
    console.log(this.form.value);
  }
}
```

`FormBuilder`の`group`で、フォームの構造をクラス側に定義しています。各項目には、初期値と検証ルール（`Validators`）を指定します。テンプレートでは、`[formGroup]`でフォーム全体を、`formControlName`で各入力を結びつけます。フォームの構造も検証も、すべてクラス側にあるため、見通しがよく、テストもしやすくなります。

Reactive Formsは、複雑なフォームに向きます。項目が多い、動的に項目が増減する、複雑な検証がある、といった場合に、その堅牢さが活きます。実務の込み入ったフォームでは、Reactiveが選ばれることが多くなります。

### フォームの検証

どちらの方式でも、入力値の検証（バリデーション）ができます。「必須である」「メールアドレスの形式である」「文字数が範囲内である」といったルールを課し、満たさなければエラーとして扱います。

Reactive Formsでは、先ほどの`Validators.required`や`Validators.email`のように、検証ルールをクラス側で指定しました。フォームやコントロールは、`valid`・`invalid`・`errors`といった状態を持ち、これをテンプレートで使って、エラーメッセージの表示や送信ボタンの制御を行います。

```html
@if (form.controls.email.invalid && form.controls.email.touched) {
  <p>正しいメールアドレスを入力してください</p>
}
```

`touched`は、その項目が一度でも操作されたかを表します。「操作された後で、かつ無効なとき」にだけエラーを出す、という自然な制御が、これで書けます。検証は、フォームの使い勝手を左右する重要な要素です。入力していないうちからエラーを出すと、利用者を不快にさせます。`touched`や`dirty`（値が変更されたか）を使い、適切なタイミングでエラーを見せる配慮が大切です。

なお、`Validators`には標準の検証がひととおり揃っています。`required`（必須）、`email`（メール形式）、`minLength`／`maxLength`（文字数）、`min`／`max`（数値の範囲）、`pattern`（正規表現）などです。標準にない独自の検証は、自作の検証関数（カスタムバリデーター）として書けます。「パスワードと確認用が一致するか」のような、複数項目にまたがる検証も、フォーム全体に対する検証関数として実装できます。

### よくあるつまずき

- **`FormsModule`と`ReactiveFormsModule`の混同**: Template-drivenは`FormsModule`、Reactiveは`ReactiveFormsModule`をimportします。使う方式に応じて、正しいほうを宣言します。
- **`formControlName`の綴り違い**: `FormGroup`で定義した名前と、テンプレートの`formControlName`が一致していないと、結びつきません。名前の綴りを確認します。
- **入力前からエラーを出す**: 検証結果をそのまま表示すると、まだ入力していない項目にもエラーが出ます。`touched`と組み合わせ、操作後にだけ表示します。
- **単純なフォームにReactiveを持ち込む**: 項目が1つ2つのフォームにまで`FormGroup`を用意すると、かえって大げさです。フォームの規模に見合った方式を選びます。逆に、複雑なフォームをTemplate-drivenで押し通すと、テンプレートが肥大化します。フォームの複雑さに応じて、方式を見直す柔軟さも大切です。

### 2つの方式の使い分け

Template-drivenとReactiveの使い分けの目安を、表に整理します。

| 観点 | Template-driven | Reactive |
|---|---|---|
| フォームの定義 | テンプレート | クラス |
| 手軽さ | 高い | やや手間 |
| 複雑なフォーム | 苦手 | 得意 |
| テスト | しにくい | しやすい |
| 動的なフォーム | 苦手 | 得意 |

一般には、単純なフォームならTemplate-driven、複雑なフォームならReactive、という使い分けが基本でした。ただし、次章で学ぶSignal Formsという新しい選択肢が加わったことで、この構図も変わりつつあります。まずは、この2方式が「テンプレート主役か、クラス主役か」で分かれることを押さえてください。

## Typed FormsとSignal Forms

前節で、Template-drivenとReactiveという2つのフォーム方式を学びました。この節では、その先の進化を扱います。ひとつは、Reactive Formsを型安全にしたTyped Forms。もうひとつは、Angular 22で安定版となった、まったく新しいSignal Formsです。

フォームは、Angularが継続的に改善を重ねてきた領域です。Typed Formsは型の穴を塞ぎ、Signal Formsは、これまで学んだSignalの考え方をフォームに持ち込みました。とくにSignal Formsは、モダンAngularのフォームの中心となることが期待される、重要な新機能です。この節では、両者の考え方と書き方を、これまでの方式と比較しながら学びます。

### Typed Forms

Reactive Formsは、長らく型の面に弱点を抱えていました。フォームから取り出した値の型が、必ずしも正確でなく、思わぬ実行時エラーを招くことがあったのです。この問題を解決したのが、Angular 14（2022年）で導入されたTyped Formsです。

Typed Formsでは、フォームの構造から、値の型が正確に導かれます。前節の`FormBuilder`による定義は、実はすでにTyped Formsとして動いています。

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

### Signal Formsの登場

Reactive Formsは堅牢ですが、RxJSベースであり、値の変化は`valueChanges`というObservableで受け取ります。第6部・第8部で見たように、モダンAngularの状態管理はSignalへ移りつつあります。フォームだけがObservableのままでは、アプリ全体の一貫性が損なわれます。

そこで登場したのが、Signal Formsです。Angular 22（2026年）で安定版になった、Signalを土台とする新しいフォームの仕組みです。フォームの状態がSignalとして表現されるため、これまで学んできたSignalの考え方が、そのままフォームに通用します。`@angular/forms/signals`から機能をimportして使います。

Signal Formsの核心は、フォームのデータを、ふつうのSignalとして持つことです。そのSignalモデルから、`form()`関数でフォームを組み立てます。

### form()でフォームを作る

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

### テンプレートとの結びつけと状態

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

### 3つの方式の位置づけ

これで、フォームの方式が3つになりました。それぞれの位置づけを整理します。

| 方式 | 中心 | 特徴 |
|---|---|---|
| Template-driven | テンプレート | 手軽。単純なフォーム向け |
| Reactive | クラス（RxJSベース） | 堅牢。従来の複雑なフォームの定番 |
| Signal Forms | Signalモデル | 型安全でSignal一貫。v22の新標準候補 |

Signal Formsは、Reactive Formsの堅牢さと型安全性を保ちながら、Signalベースである点で、モダンAngularとの一貫性に優れます。Angularは、既存のReactive Formsからの段階的な移行も見据えており、両者を併存させながら移していけるよう配慮されています。

本書では、新規に複雑なフォームを作るなら、Signalベースで書けるSignal Formsを第一の選択肢として推奨します。ただし、Signal Formsは登場して間もないため、既存プロジェクトの多くはReactive Formsで書かれています。当面は、両方を理解しておくことが実務上は重要です。単純なフォームであれば、Template-drivenも引き続き有効です。

### なぜSignal Formsが重要なのか

Signal Formsが重要なのは、フォームの状態を、アプリケーションのほかの状態と同じ「Signal」という土俵に載せる点にあります。第6部で見たように、モダンAngularは状態管理をSignalに統一する方向へ進んでいます。ところが、フォームだけがRxJSベースのままだと、フォームの値をほかのSignalと組み合わせるたびに、変換（`toSignal()`など）が必要でした。

Signal Formsでは、フォームの値も検証状態も、最初からSignalです。`computed()`でフォームの値から別の値を導いたり、`effect()`でフォームの変化に反応したりが、変換なしに、そのまま書けます。フォームが、アプリの状態管理に自然に溶け込むのです。これは、単なる書き方の違いではなく、アプリケーション全体の一貫性という、より大きな利点につながります。第6部から一貫して見てきた「状態はSignalで持つ」という流れの、フォームにおける到達点が、Signal Formsだといえます。

### よくあるつまずき

- **フォームの状態の読み方を間違える**: Signal Formsでは、項目を関数として呼んでから状態を読みます。`loginForm.email().value()`のように、二段階になる点に注意します。
- **方式を混同する**: Template-driven（`ngModel`）、Reactive（`formControlName`）、Signal Forms（`[formField]`）は、それぞれ結びつけ方が違います。1つのフォームでは、方式を統一します。
- **新しさだけで飛びつく**: Signal Formsは有望ですが、既存プロジェクトがReactiveで統一されているなら、無理に混在させず、方針を決めて選びます。
- **検証を検証関数の外に書く**: Signal Formsの検証は、スキーマ関数の中で`required(path.email)`のように宣言します。テンプレート側で条件分岐を駆使して検証を模倣すると、Signal Formsの利点が薄れます。ルールはスキーマに集約します。

## まとめ

- Angularには伝統的に、Template-drivenとReactiveの2つのフォーム方式があります
- Template-drivenは`ngModel`を使い、テンプレートを主役に手軽に組み立てます
- Reactiveは`FormControl`・`FormGroup`をクラスで定義し、堅牢に組み立てます
- Typed Formsは、Reactive Formsの値に正確な型を与える仕組みで、v14以降は標準です
- Signal Formsは、v22で安定化した、Signalを土台とする新しいフォーム方式です
- `form(model, スキーマ関数)`でフォームを作り、`[formField]`で入力欄に結びつけます

次章では、サーバーとデータをやり取りするHTTP通信を学びます。
