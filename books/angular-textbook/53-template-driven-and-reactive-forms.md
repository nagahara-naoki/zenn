---
title: "第43章 Template-driven FormsとReactive Forms"
---

フォームは、利用者からの入力を受け取る、アプリケーションの重要な入り口です。名前の入力、ログイン、検索、設定の変更。これらはすべてフォームです。Angularは、フォームを扱うための仕組みを複数用意しています。この章では、伝統的な2つの方式、Template-driven FormsとReactive Formsを学びます。

この2つは、長らくAngularのフォームの二本柱でした。Template-drivenは手軽で、Reactiveは堅牢です。それぞれに向き不向きがあり、どちらを使うかは、フォームの複雑さによって判断します。次章では、これらに加えてSignal Formsという新しい方式も学びますが、まずは既存の2方式を理解することが、フォーム全体を捉える土台になります。

:::message
**この章で学ぶこと**

- Angularにおける2つのフォーム方式
- Template-driven Formsの書き方
- Reactive Formsの書き方
- 2つの方式の使い分け
:::

## 2つのフォーム方式

Angularには、伝統的に2つのフォーム方式があります。

- **Template-driven Forms**: テンプレート側を主役に、フォームを組み立てる方式です。`ngModel`を使い、テンプレートに書いた指定から、Angularがフォームの構造を読み取ります。手軽さが特徴です。
- **Reactive Forms**: クラス側を主役に、フォームを組み立てる方式です。`FormControl`や`FormGroup`をクラスで定義し、それをテンプレートに結びつけます。構造が明示的で、堅牢さが特徴です。

大きな違いは、「フォームの定義が、テンプレートとクラスのどちらにあるか」です。Template-drivenはテンプレートに、Reactiveはクラスに、フォームの中心があります。順に見ていきましょう。

## Template-driven Forms

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

## Reactive Forms

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

## フォームの検証

どちらの方式でも、入力値の検証（バリデーション）ができます。「必須である」「メールアドレスの形式である」「文字数が範囲内である」といったルールを課し、満たさなければエラーとして扱います。

Reactive Formsでは、先ほどの`Validators.required`や`Validators.email`のように、検証ルールをクラス側で指定しました。フォームやコントロールは、`valid`・`invalid`・`errors`といった状態を持ち、これをテンプレートで使って、エラーメッセージの表示や送信ボタンの制御を行います。

```html
@if (form.controls.email.invalid && form.controls.email.touched) {
  <p>正しいメールアドレスを入力してください</p>
}
```

`touched`は、その項目が一度でも操作されたかを表します。「操作された後で、かつ無効なとき」にだけエラーを出す、という自然な制御が、これで書けます。検証は、フォームの使い勝手を左右する重要な要素です。入力していないうちからエラーを出すと、利用者を不快にさせます。`touched`や`dirty`（値が変更されたか）を使い、適切なタイミングでエラーを見せる配慮が大切です。

なお、`Validators`には標準の検証がひととおり揃っています。`required`（必須）、`email`（メール形式）、`minLength`／`maxLength`（文字数）、`min`／`max`（数値の範囲）、`pattern`（正規表現）などです。標準にない独自の検証は、自作の検証関数（カスタムバリデーター）として書けます。「パスワードと確認用が一致するか」のような、複数項目にまたがる検証も、フォーム全体に対する検証関数として実装できます。

## よくあるつまずき

- **`FormsModule`と`ReactiveFormsModule`の混同**: Template-drivenは`FormsModule`、Reactiveは`ReactiveFormsModule`をimportします。使う方式に応じて、正しいほうを宣言します。
- **`formControlName`の綴り違い**: `FormGroup`で定義した名前と、テンプレートの`formControlName`が一致していないと、結びつきません。名前の綴りを確認します。
- **入力前からエラーを出す**: 検証結果をそのまま表示すると、まだ入力していない項目にもエラーが出ます。`touched`と組み合わせ、操作後にだけ表示します。
- **単純なフォームにReactiveを持ち込む**: 項目が1つ2つのフォームにまで`FormGroup`を用意すると、かえって大げさです。フォームの規模に見合った方式を選びます。逆に、複雑なフォームをTemplate-drivenで押し通すと、テンプレートが肥大化します。フォームの複雑さに応じて、方式を見直す柔軟さも大切です。

## 2つの方式の使い分け

Template-drivenとReactiveの使い分けの目安を、表に整理します。

| 観点 | Template-driven | Reactive |
|---|---|---|
| フォームの定義 | テンプレート | クラス |
| 手軽さ | 高い | やや手間 |
| 複雑なフォーム | 苦手 | 得意 |
| テスト | しにくい | しやすい |
| 動的なフォーム | 苦手 | 得意 |

一般には、単純なフォームならTemplate-driven、複雑なフォームならReactive、という使い分けが基本でした。ただし、次章で学ぶSignal Formsという新しい選択肢が加わったことで、この構図も変わりつつあります。まずは、この2方式が「テンプレート主役か、クラス主役か」で分かれることを押さえてください。

## まとめ

- Angularには伝統的に、Template-drivenとReactiveの2つのフォーム方式があります
- Template-drivenは`ngModel`を使い、テンプレートを主役に手軽に組み立てます
- Reactiveは`FormControl`・`FormGroup`をクラスで定義し、堅牢に組み立てます
- どちらも`Validators`による検証ができ、`valid`・`touched`などの状態を持ちます
- 単純なフォームはTemplate-driven、複雑なフォームはReactiveが基本の使い分けです

次章では、Reactive Formsを型安全にするTyped Formsと、Angular 22の新しいSignal Formsを学びます。
