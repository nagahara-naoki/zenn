---
title: "用語集と目的別索引"
---

Web Componentsには、似た名前の概念がいくつも登場します。実装中に迷ったときは、この章で言葉の関係を確認してください。前半は困りごとから読む章を引く索引、後半は用語の定義です。

## 目的から章を探す

| 知りたいこと・困っていること | 読む章 |
|---|---|
| Web Componentsが何を解決するのか | Web Componentsとは何か |
| 独自要素がいつ有効になるのか | 要素の定義とアップグレード |
| DOMから外したらリスナーが重複した | ライフサイクルコールバック |
| 属性とプロパティのどちらを使うか | 属性とプロパティ |
| 状態更新でフォーカスが失われる | 状態とレンダリング |
| ページのCSSが内部へ当たってしまう | Shadow DOMとは何か / Shadow DOMのスタイリング |
| 利用者のHTMLを内部へ表示したい | TemplateとSlot |
| Shadow DOMの外へ操作を通知したい | CustomEventとイベント伝播 |
| 外からテーマを変更できるようにしたい | Shadow DOMのスタイリング / テーマとスタイルシートの共有 |
| 公開するプロパティやイベントを決めたい | 公開APIの設計 |
| キーボードと支援技術へ対応したい | アクセシビリティとフォーカス |
| 独自入力を`FormData`へ含めたい | ElementInternalsとフォーム連携 |
| XSS対策を確認したい | セキュリティとXSS対策 |
| 多数の要素を描画すると遅い | パフォーマンスの計測と改善 |
| タグ名やイベントに型を付けたい | TypeScriptとの統合 |
| 実ブラウザでテストしたい | 実ブラウザでのテスト |
| npmで配布したい | npmパッケージ化とドキュメント |
| React、Vue、Angularから使いたい | フレームワークとの連携 |
| JavaScript実行前にShadow DOMを表示したい | Declarative Shadow DOMとSSR |
| 新しいAPIの採用可否を判断したい | Custom StatesとScoped Registries |
| 手作業の描画更新を減らしたい | Litとは何か |
| 要素をまとめて1つのUIへ組み上げたい | タスク管理UIを完成させる |

## Custom ElementとCustom Elements

Custom Elementは、開発者が定義した独自のHTML要素です。`<task-item>`のように、名前にはハイフンが必要です。

Custom Elementsは、要素名とJavaScriptクラスを登録し、要素の生成やupgrade、ライフサイクルを扱う仕組み全体を指します。個々の要素と、仕組みの名前を区別してください。

Autonomous Custom Elementは、`HTMLElement`を直接継承する独自要素です。本書ではこちらを基本にします。Customized Built-in Elementは`HTMLButtonElement`などを拡張し、`is`属性を使う別の形式です。

## Custom Element Registry・定義・upgrade

Custom Element Registryは、要素名とクラスの対応を保持するブラウザの登録簿です。通常はページ全体で共有する`customElements`を使います。

定義とは、`customElements.define()`で名前とクラスを結び付けることです。同じ名前は原則として再定義できません。

upgradeは、定義前からDOMに存在していた要素へ、登録されたクラスの振る舞いが加わる処理です。HTMLの解析とJavaScriptの読み込みには時間差があるため、Custom Elementは定義前にも存在し得ます。

## ホスト・Light DOM・Shadow Root・Shadow Tree

Shadow Hostは、Shadow Rootが取り付けられた要素です。多くの場合、独自要素そのものがホストを兼ねます。

Light DOMは、ホストの子として利用者が書いたノードです。`<task-item><span>タスク名</span></task-item>`なら、`span`はLight DOMにあります。

Shadow Rootは、ホストと内部ツリーを結ぶ入口です。`attachShadow()`の戻り値であり、`mode: 'open'`ならホストの`shadowRoot`から参照できます。

Shadow Treeは、Shadow Rootの内側にあるDOMツリーです。内部のチェックボックスやレイアウト要素を置けます。ページ側の通常のCSSセレクターは、原則としてこの境界を越えません。

これらは表示上、合成された1つのツリーとして扱われます。Slotへ割り当てられたLight DOMは、所有場所を変えずにShadow Tree内の位置へ表示されます。

## TemplateとSlot

`<template>`は、すぐには表示しないDOMのひな形を保持する要素です。内容は`content`プロパティの`DocumentFragment`にあり、複製してShadow Rootなどへ追加します。

`<slot>`は、利用者がLight DOMに書いた内容の表示位置を示します。Slotはノードを内部へコピーしません。コンポーネント作者が内部構造を所有し、利用者が差し込む内容を所有するという分担を作ります。

名前付きSlotは、`name`と`slot`属性で対応させます。Slot名は利用者のHTMLに残るため、公開後に変更すると破壊的変更になります。

## 属性・プロパティ・reflection

属性はHTMLに文字列として書ける設定です。ページのソースや開発者ツールから確認でき、CSSの属性セレクターでも使えます。

プロパティは、実行中のDOMオブジェクトが持つ値です。文字列だけでなく、真偽値、配列、オブジェクト、関数も保持できます。

reflectionは、同じ状態を属性とプロパティの間で反映することです。すべての値を双方向に同期する必要はありません。HTMLから設定する意味があり、外から観測する価値がある状態だけを反映します。

## ライフサイクル

ライフサイクルコールバックは、Custom Elementと文書の関係が変わったときにブラウザから呼ばれます。

- `constructor()`はインスタンス生成時の土台作りに使う
- `connectedCallback()`は文書へ接続したときの購読や表示開始に使う
- `disconnectedCallback()`は購読解除や停止に使う
- `attributeChangedCallback()`は監視対象の属性変更を状態へ反映する
- `adoptedCallback()`は別のDocumentへ移ったときに使う

接続と切断は何度でも起こり得ます。切断を永久破棄と決めつけず、再接続しても状態が壊れないようにします。

## CustomEvent・bubbles・composed・retargeting

`CustomEvent`は、独自のデータを`detail`へ入れられるDOMイベントです。内部要素の参照ではなく、状態更新に必要なIDや値だけを渡します。

`bubbles`はイベントが祖先へ伝播するかを決めます。`composed`はShadow DOMの境界を越えられるかを決めます。外部へ公開する操作イベントでは、両方を`true`にすることが一般的です。

retargetingは、イベントがShadow DOMの境界を越えるとき、外から見える`event.target`がホストへ置き換わる仕組みです。利用側が内部要素へ依存しにくくなります。

## CSS Custom PropertiesとCSS Parts

CSS Custom Propertiesは、色や余白などの値を外から渡す接点です。`--task-item-accent-color`のように、変更してよい意味を名前にします。

CSS Partsは、Shadow Tree内の特定要素を外部CSSへ公開する仕組みです。内部要素へ`part`属性を付け、利用側は`::part()`で選びます。Part名は内部クラス名ではなく、維持する意思のあるスタイルAPIです。

## ElementInternalsとForm-associated Custom Element

`ElementInternals`は、Custom Elementがフォーム、アクセシビリティ、Custom Statesなどのブラウザ機能へ参加するための内部APIです。`attachInternals()`で取得します。

Form-associated Custom Elementは、フォーム送信、制約検証、リセット、無効化へ参加する独自要素です。`static formAssociated = true`を宣言し、`setFormValue()`で送信値をブラウザへ渡します。

## Declarative Shadow DOMとhydration

Declarative Shadow DOMは、`<template shadowrootmode="open">`を使い、HTMLだけでShadow Rootを宣言する仕組みです。サーバーから内部構造を返せるため、JavaScript実行前の表示に利用できます。

hydrationは、サーバーが返したHTMLへクライアント側の動作を接続する処理です。独自要素の場合は、既存のShadow Rootを作り直さずに、イベントリスナーと状態同期だけを追加します。

## Custom StatesとScoped Custom Element Registries

Custom Statesは、内部状態の名前を`:state()`疑似クラスへ公開する仕組みです。HTML属性として利用者に設定させたくない状態でも、CSSから選択できます。状態名はCSS契約になるため、privateな情報ではありません。

Scoped Custom Element Registriesは、要素名とクラスの対応をShadow Rootごとに持つ仕組みです。グローバルな名前の衝突を避けられますが、対象ブラウザの対応状況を確認し、基本機能の前提にするかを判断します。

## Lit

LitはWeb Componentsを置き換えるフレームワークではありません。Custom Element、Shadow DOM、属性、イベント、Slotの上で、状態からDOMを更新する定型処理を減らすライブラリです。

ネイティブ実装からLitへ移行しても、タグ名、属性、プロパティ、イベント、Slot、CSS Partsを保てば、利用側の契約は変えずに済みます。

## 更新情報と問い合わせ

本文の仕様・互換性は2026年8月9日に確認しました。ブラウザ対応が重要な機能は、各章の参考資料と対象ブラウザで再確認してください。

誤りや分かりにくい箇所は、Zennのコメントで知らせてください。仕様変更と読者からの指摘をもとに改訂します。
