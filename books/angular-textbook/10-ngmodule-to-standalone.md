---
title: "NgModuleからStandalone Componentへ"
---

前章では、Componentのテンプレートで別の部品を使うために`imports`へ宣言する、という書き方を見ました。この`imports`は、Standalone Componentという仕組みの一部です。この章では、そのStandalone Componentが登場する前にどうしていたのか、つまり従来のNgModuleと比較しながら、なぜいまの形になったのかを掘り下げます。

現在の新規開発では、NgModuleを書く場面はほとんどありません。しかし、少し前のプロジェクトや、Web上の記事にはNgModuleが数多く登場します。両者の関係を理解しておくと、古いコードを読むときにも、移行を進めるときにも役立ちます。

:::message
**この章で学ぶこと**

- NgModuleがどのような仕組みだったか
- NgModuleが抱えていた課題
- Standalone Componentによる解決
- 既存プロジェクトを移行する方法
:::

## NgModuleとは何だったか

かつてのAngularでは、Component・Directive・Pipeを、単体で使うことができませんでした。これらは必ず、NgModuleという入れ物に登録する必要がありました。NgModuleは、`@NgModule`デコレーターを付けたクラスで、次のような設定を持ちます。

```ts:旧来の書き方（NgModule）
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Greeting } from './greeting';

@NgModule({
  declarations: [Greeting],
  imports: [CommonModule],
  exports: [Greeting],
})
export class GreetingModule {}
```

それぞれの設定には、次の役割があります。

- **declarations**: このモジュールに属するComponent・Directive・Pipeを登録する
- **imports**: このモジュールが必要とする、別のモジュールを取り込む
- **exports**: 外部のモジュールに公開する部品を指定する
- **providers**: DIで注入するServiceを登録する

NgModuleは、関連する部品をひとまとめにして、どれを外部に公開し、どれを内部にとどめるのかを管理する仕組みでした。中規模以上のアプリケーションを整理するうえでは、たしかに役立ちました。

## NgModuleが抱えていた課題

一方で、NgModuleには扱いにくさもありました。

まず、Componentを1つ作るたびに、それをどこかのNgModuleの`declarations`に登録しなければなりませんでした。小さな部品を1つ足すだけでも、Component本体とNgModuleの2か所を触ることになります。初学者にとっては、「なぜこの登録がいるのか」が分かりにくく、理解の壁になっていました。

次に、ある部品を使いたいとき、それがどのNgModuleに属していて、どのモジュールを`imports`すればよいのかを追う必要がありました。この依存関係の見通しの悪さも、開発の負担になっていました。

つまりNgModuleは、部品を束ねる便利さと引き換えに、記述の手間と理解の難しさを生んでいたのです。

## Standalone Componentによる解決

この課題に応えて登場したのが、Standalone Componentです。Standalone Componentは、NgModuleという入れ物を必要とせず、単体で成立します。自身が必要とする部品を、`@Component`の`imports`に直接宣言するのが特徴です。

```ts:src/app/greeting.ts
import { Component } from '@angular/core';
import { UserIcon } from './user-icon';

@Component({
  selector: 'app-greeting',
  imports: [UserIcon],
  template: `
    <app-user-icon />
    <p>こんにちは</p>
  `,
})
export class Greeting {}
```

必要な依存が、Componentの中に閉じて宣言されているため、そのComponentが何に依存しているのかがひと目でわかります。NgModuleという中間層がなくなり、部品とその依存の関係がまっすぐに見えるようになりました。

Standalone Componentは、Angular 14（2022年）でプレビューとして登場し、Angular 17（2023年11月）で新規プロジェクトの標準的な生成形式になりました。さらにAngular 19（2024年11月）では、`standalone: true`という指定すら不要になり、Standaloneであることが暗黙のデフォルトになっています。そのため、現在のコードでは`standalone: true`という記述を書きません。

:::message
少し前のプロジェクトでは、`@Component`に`standalone: true`と明記されていることがあります。これはStandaloneであることを示す当時の書き方で、Angular 19以降では省略できます。逆に`standalone: false`と書かれている場合は、NgModuleに登録して使う従来のComponentを表します。
:::

Standaloneには、見通しのよさ以外の利点もあります。各Componentが自分の依存を把握しているため、必要になったときにだけ読み込む「遅延読み込み」がしやすくなります。結果として、最初に読み込むコードの量を減らし、アプリの初期表示を速くしやすくなります。遅延読み込みは、第7部のルーティングで詳しく扱います。

## 新旧のコードを比べる

同じ「Greetingを画面に表示する」構成を、NgModule時代とStandalone時代で比べてみます。まずNgModule時代は、Componentのほかに、それを束ねるモジュールが必要でした。

```ts:旧来の書き方（NgModule時代）
// greeting.ts と greeting.module.ts の2ファイルが必要
@NgModule({
  declarations: [Greeting],
  exports: [Greeting],
})
export class GreetingModule {}
```

Standalone時代では、モジュールが不要になり、Component1つで完結します。

```ts:src/app/greeting.ts（現在の書き方）
@Component({
  selector: 'app-greeting',
  imports: [/* 必要な部品 */],
  templateUrl: './greeting.html',
})
export class Greeting {}
```

ファイル数が減り、依存関係がComponentの中に集約されているのがわかります。

## モジュールから関数へ

NgModuleは、部品を束ねるだけでなく、アプリ全体の設定を登録する役割も担っていました。この役割も、モダンAngularでは形を変えています。

たとえば、ルーティングを使うには、かつては`RouterModule.forRoot(routes)`をNgModuleの`imports`に登録していました。現在は、第4章で見たように、`app.config.ts`で`provideRouter(routes)`という関数を使います。同じように、HTTP通信も`HttpClientModule`のimportから`provideHttpClient()`へと変わりました。「モジュールをimportする」書き方から、「`provide〜`という関数を並べる」書き方へ移ったのです。

```ts:旧来の書き方（NgModuleでの設定登録）
@NgModule({
  imports: [RouterModule.forRoot(routes), HttpClientModule],
})
export class AppModule {}
```

```ts:src/app/app.config.ts（現在の書き方）
export const appConfig: ApplicationConfig = {
  providers: [provideRouter(routes), provideHttpClient()],
};
```

また、`*ngIf`や`*ngFor`を使うために必要だった`CommonModule`のimportも、現在は不要です。条件分岐や繰り返しは、`@if`や`@for`という組み込みの構文に置き換わり、importなしで使えるようになりました。これらの新しい構文は、第3部で詳しく扱います。

かつては、よく使う部品をまとめた`SharedModule`を作り、各所でimportする手法も一般的でした。Standaloneでは、必要なComponentやDirectiveを、使う側が直接`imports`に書きます。共有のための中間モジュールを設けなくても、部品を個別に取り込めます。

## 既存プロジェクトを移行する

NgModuleで書かれた既存のプロジェクトは、公式の移行ツール（スキマティクス）を使って、Standaloneへ自動で変換できます。次のコマンドで実行します。

```bash
ng generate @angular/core:standalone
```

この移行は、大きく3つの段階で進みます。

1. すべてのComponent・Directive・PipeをStandaloneにし、必要な依存を各Componentの`imports`へ移す
2. 不要になったNgModule（部品を再公開するだけのもの）を削除する
3. 起動処理を`NgModule`ベースから`bootstrapApplication`へ置き換える

自動で判断できない部分には印（TODOコメント）が残されるため、そこは手作業で確認します。すべてを一度に変換しようとせず、段階的に進められるのも、この移行ツールの特徴です。NgModuleとStandaloneは共存できるため、大きなプロジェクトでも、機能ごとに少しずつ移行を進められます。一度にすべてを書き換える必要はありません。

:::message
移行ツールを実行する前に、変更をコミットしておくと安心です。自動変換の結果を差分で確認しながら、段階的に進めましょう。うまく変換できなかった箇所は、残されたTODOコメントを手がかりに、手作業で対応します。
:::

## まとめ

- NgModuleは、Component・Directive・Pipeを束ねて管理する従来の仕組みでした
- 便利な一方で、登録の手間や依存関係の見通しの悪さという課題を抱えていました
- Standalone Componentは、必要な依存を`imports`に直接宣言し、単体で成立します
- Angular 19以降、`standalone: true`の指定は不要で、Standaloneが暗黙のデフォルトです
- 既存プロジェクトは、公式の移行ツールで段階的にStandaloneへ変換できます
- **新規開発ではStandalone Componentを使うのが現在の標準です。NgModuleは既存コードを読み解くための知識として押さえておきましょう**

次章では、Standalone Componentを組み合わせるうえで役立つ、コンテンツ投影と子要素の参照（クエリ）を学びます。
