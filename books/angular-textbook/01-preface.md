---
title: "本書の読み方 — 対象読者と読み進め方"
---

本書は、Angularをこれから学ぶ初学者から、実務でAngularを使いこなす上級者までを対象にした教科書です。Angularの機能を表面的に紹介するのではなく、「なぜそう動くのか」「どう設計すべきか」という仕組みと判断の根拠まで踏み込んで解説します。

Angularは、登場以来、長い時間をかけて大きく姿を変えてきました。とくにここ数年のSignals・Standalone Component・Zonelessといった変化は、コードの書き方そのものを刷新するものでした。その結果、Web上には新しい書き方と古い書き方が混在し、学習を始めた人がどちらを信じればよいのか迷いやすい状況が生まれています。本書はこの混乱を解消することを、ひとつの大きな目的としています。

## 本書が目指すもの

本書には、次の3つの方針があります。

- **網羅的であること**: Component・Directive・Pipe・Service・Dependency Injection（DI）・変更検知（Change Detection）・Router・RxJS・Forms・状態管理・テストまで、Angularアプリケーションを構築するために必要な要素を体系的に扱います。
- **深く解説すること**: APIの使い方だけでなく、その背後にある仕組みを説明します。たとえばSignalsは、単なる状態管理の道具としてではなく、変更検知の仕組みと結びつけて解説します。
- **新旧を比較すること**: 現在の標準的な書き方を主軸に置きつつ、それが以前はどう書かれていたのかを対比します。これは古い書き方を否定するためではなく、既存コードを読み解く力を養い、変化の理由を理解するためです。

## 対象読者

本書は、次のような読者を想定しています。

- HTML・CSS・JavaScriptの基礎を理解している方
- これからAngularを学び始める方
- 他のフレームワークからAngularに移行する方
- 古い情報と新しい情報の違いに戸惑っている方
- 実務でAngularを使っており、設計判断の根拠を固めたい方

TypeScriptについては、Angularを書くうえで必要な範囲を第5章「Angularで使うTypeScript」で改めて解説します。TypeScriptに不慣れでも読み進められます。

## 本書のレベル設計

本書は全11部で構成され、進むにつれてレベルが上がる作りになっています。

| レベル | 対応する部 | 到達目標 |
|---|---|---|
| 基礎 | 第1部〜第4部 | Componentを組み合わせて画面を作れる |
| 中級 | 第5部〜第9部 | 仕組みを理解し、機能を適切に設計・実装できる |
| 上級 | 第10部〜第11部 | 状態管理やアーキテクチャの設計判断ができる |

各部のねらいと収録する章は、次の「本書の構成」で一覧できます。

## 本書の構成

本書は、序章と11の部から成ります。各部のねらいと、収録する章は次のとおりです。

**第1部 Angularを始める**（基礎）: Angularとは何か／Angularの歴史とモダンAngularへの変化／開発環境とAngular CLI／プロジェクト構成とAngularアプリが起動するまで

**第2部 Componentの基礎**（基礎）: Angularで使うTypeScript／Componentとは何か／NgModuleからStandalone Componentへ／コンテンツ投影とクエリ／スタイリングとView Encapsulation／Componentの分割と責務

**第3部 テンプレート・Directive・Pipe**（基礎）: データバインディングとイベント処理／条件分岐と繰り返し表示の新旧比較／Directiveとは何か／属性Directiveを作成する／構造Directiveとng-templateの仕組み／Pipeとテンプレートの再利用

**第4部 Component間の状態伝播**（基礎）: Angularのデータフロー／`@Input`から`input()`へ／`@Output`から`output()`へ／双方向バインディングと`model()`／ライフサイクルと入力値の変更検知

**第5部 ServiceとDependency Injection**（中級）: ServiceとComponentの責務／Dependency Injectionとは何か／Constructor Injectionから`inject()`へ／ProviderとInjectorの階層

**第6部 変更検知とSignals**（中級）: Angularの変更検知を理解する／Default Change DetectionとOnPush／Zone.jsとNgZone時代のAngular／`signal()`・`computed()`・`effect()`による状態管理／NgZoneからSignals・Zonelessへ

**第7部 Angular Router**（中級）: SPAとAngular Router／`RouterModule`から`provideRouter()`へ／ページ遷移とルートパラメーター／ネストしたRouteとレイアウト設計／Lazy LoadingとFeature分割／Route GuardとResolverの新旧比較

**第8部 RxJS**（中級）: Observableとリアクティブプログラミング／SubscriptionとObservableのライフサイクル／RxJS Operatorsと非同期処理／Subject・BehaviorSubject・ReplaySubject／RxJSとSignalsの共存／RouterとRxJS・Signals・状態管理

**第9部 FormsとHTTP通信**（中級）: Template-driven FormsとReactive Forms／Typed FormsとSignal Forms／HttpClientとAPI通信／HttpClientから`resource()`・`httpResource()`へ／Interceptor・認証・エラー・ローディング設計

**第10部 Angularの状態管理とNgRx**（上級）: 状態を分類する／BehaviorSubjectによるStore Service／SignalsによるStore Service／NgRxとReduxパターン／Action・Reducer・Selectorの設計／EffectsとRxJSによる非同期処理／Entity・Facade・Router Storeによる実務設計／NgRx SignalStoreとNgRx Storeの使い分け

**第11部 実務的なAngular開発**（上級）: アーキテクチャ設計／テストの基礎（TestBed・Harness・Vitest）／RxJS・NgRx・非同期処理のテスト戦略／セキュリティ（XSS対策）／アクセシビリティと国際化（i18n）／パフォーマンス最適化と`@defer`／SSRとHydration／モダンAngularへの移行戦略

## 本書の読み進め方

本書は、第1部から順に読むことを基本としています。上の「本書の構成」で各部のねらいと収録する章を一覧できるので、いま自分がどの位置を学んでいるのかを把握しながら、必要に応じて読む順番を調整してください。

## 本書の技術基準

本書は、2026年6月にリリースされたAngular v22を基準に解説します。この世代のAngularでは、次の内容が標準になりました。

- Signal Formsが安定版となり、実務で利用できるようになった
- Zoneless（Zone.jsに依存しない変更検知）が新規プロジェクトのデフォルトになった
- 新規Componentの変更検知がOnPushをデフォルトとするようになった
- テストの標準ツールがVitestになった

本書では、これらを「新しく登場した選択肢」ではなく「現在の標準」として扱います。旧来の書き方は、既存コードを読み解き、移行を進めるための知識として解説します。

バージョンによって挙動が変わる機能については、「v17で導入され」「v22で安定化した」のように、その都度バージョンを明示します。

## 本書の表記ルール

本書では、読みやすさのために次の表記を用います。

API名・クラス名・ファイル名などコード上の要素は、`signal()`や`@Input`のように等幅フォントで示します。サンプルコードには、対応するファイル名を添えます。

```ts:src/app/user-profile.ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-user-profile',
  template: `<p>{{ userName() }}</p>`,
})
export class UserProfile {
  userName = input.required<string>();
}
```

旧来の書き方を示すコードには、「旧」であることを明記します。補足や注意点は、次のような囲み枠で示します。

:::message
このような枠には、本文の補足や注意点を記載します。
:::

発展的な内容や長いコードは、折りたたみの中に収めることがあります。

## まとめ

- 本書はAngularの機能を、仕組みと設計判断の根拠まで掘り下げて解説します
- 現在の標準的な書き方を主軸に、旧来の書き方と対比しながら学びます
- 全11部を通じて、基礎から状態管理・アーキテクチャ設計まで段階的に扱います
- 技術基準は、2026年6月にリリースされたAngular v22です

それでは、Angularの学習を始めましょう。次の第1部では、そもそもAngularとは何か、どのようなフレームワークなのかを解説します。
