---
title: "条件分岐と繰り返し表示の新旧比較"
---

前章では、クラスの値をテンプレートに表示する方法を学びました。しかし実際の画面では、「ログイン済みなら名前を、未ログインならログインボタンを表示する」「商品の配列を一覧として並べる」といった、条件や繰り返しに応じた表示が欠かせません。

Angularには、テンプレートの中で条件分岐や繰り返しを書くための制御フロー構文が用意されています。現在の標準は`@if`・`@for`・`@switch`という組み込み構文です。一方、少し前のコードでは`*ngIf`・`*ngFor`という別の書き方が使われていました。この章では、新しい制御フローを主に、旧来の書き方と比較しながら学びます。両方を知っておくと、既存コードを読む力も身につきます。

:::message
**この章で学ぶこと**

- `@if`・`@for`・`@switch`による制御フロー
- `@for`の`track`と、繰り返しのコンテキスト変数
- 旧来の`*ngIf`・`*ngFor`との違い
- なぜ組み込み制御フローへ変わったのか
:::

## @ifによる条件分岐

条件に応じて表示を切り替えるには、`@if`を使います。JavaScriptの`if`文によく似た書き方です。

```html
@if (isLoggedIn()) {
  <p>こんにちは、{{ userName() }}さん</p>
} @else {
  <button (click)="login()">ログイン</button>
}
```

`@if`の丸括弧に条件式を書き、真のときは最初のブロックを、偽のときは`@else`のブロックを表示します。条件が3つ以上に分かれるときは、`@else if`をはさみます。

```html
@if (status() === 'loading') {
  <p>読み込み中</p>
} @else if (status() === 'error') {
  <p>エラーが発生しました</p>
} @else {
  <p>{{ data() }}</p>
}
```

条件式の結果を、ブロックの中で使い回したいこともあります。その場合は`as`で名前を付けられます。

```html
@if (user() as u) {
  <p>{{ u.name }}（{{ u.email }}）</p>
}
```

`user()`の値を`u`という名前で受け、ブロック内で繰り返し呼び出さずに参照できます。Signalの呼び出しを1回にまとめられるため、見通しがよくなります。

## @forによる繰り返し

配列などを繰り返し表示するには、`@for`を使います。ここで重要なのが、`track`の指定が必須である点です。

```html
@for (product of products(); track product.id) {
  <app-product-card [product]="product" />
} @empty {
  <p>商品がありません</p>
}
```

`product of products()`で配列の各要素を取り出し、`track product.id`で「各要素を何で見分けるか」を指定します。`@empty`ブロックは、配列が空のときに表示される内容です。一覧が空のときのメッセージを、自然に書けます。

`track`は、Angularが要素の増減や並び替えを効率よく画面へ反映するための目印です。商品の`id`のような、要素を一意に識別できる値を指定します。適切な`track`があると、変化のあった要素だけを更新でき、描画のむだを抑えられます。一意なIDがない場合は、`track $index`のように繰り返しの位置を使うこともできますが、可能なら意味のあるIDを使うのが望ましい書き方です。

`@for`の中では、繰り返しの状態を表すコンテキスト変数を使えます。

| 変数 | 意味 |
|---|---|
| `$index` | 0から始まる要素の位置 |
| `$count` | 要素の総数 |
| `$first` / `$last` | 最初／最後の要素か |
| `$even` / `$odd` | 位置が偶数／奇数か |

```html
@for (item of items(); track item.id; let i = $index) {
  <p>{{ i + 1 }}番目: {{ item.name }}</p>
}
```

`let i = $index`のように別名を付けて使うこともできます。行番号の表示や、偶数行だけ背景色を変える、といった処理に役立ちます。

## @switchによる分岐

とりうる値が決まっている分岐には、`@switch`が向いています。JavaScriptの`switch`文に対応します。

```html
@switch (role()) {
  @case ('admin') {
    <app-admin-panel />
  }
  @case ('member') {
    <app-member-view />
  }
  @default {
    <app-guest-view />
  }
}
```

`role()`の値に応じて、一致する`@case`のブロックを表示します。どれにも一致しなければ`@default`が使われます。比較は厳密等価（`===`）で行われ、`@if`の`@else if`を連ねるより意図が明確になります。

## 旧来の書き方との比較

これらの組み込み制御フローは、Angular 17（2023年）で導入され、Angular 18（2024年）で安定版になりました。それ以前は、`*ngIf`・`*ngFor`・`*ngSwitch`というディレクティブを使っていました。同じ条件分岐を、旧来の書き方で示します。

```html
<!-- 旧来の書き方（*ngIf） -->
<p *ngIf="isLoggedIn(); else loginBlock">こんにちは</p>
<ng-template #loginBlock>
  <button (click)="login()">ログイン</button>
</ng-template>
```

```html
<!-- 旧来の書き方（*ngFor） -->
<app-product-card
  *ngFor="let product of products(); trackBy: trackById"
  [product]="product"
/>
```

`*ngFor`で使う`trackBy`は、`track`と同じ役割を果たしますが、コンポーネント側に関数を用意する必要がありました。

```ts
// 旧来の書き方：trackBy 関数をクラスに定義
protected trackById(index: number, product: Product): number {
  return product.id;
}
```

新しい`@for`では、`track product.id`と式を直接書けるため、この関数が不要になりました。`@else`の分岐先を`ng-template`で別途用意する必要もなく、記述量が減っています。

## なぜ組み込み制御フローへ変わったのか

新しい制御フローには、旧来の書き方に対して明確な利点があります。

- **`CommonModule`のimportが不要**: `*ngIf`・`*ngFor`は`CommonModule`をimportして初めて使えました。組み込み構文は、importなしでどのテンプレートでも使えます。
- **読みやすさ**: `@if`・`@for`はJavaScriptの構文に近く、条件と繰り返しの構造が一目で追えます。`*`記法や`ng-template`の知識がなくても書けます。
- **パフォーマンス**: 組み込み構文はAngularが最適化しやすく、とくに`@for`は差分更新の効率が改善されています。
- **型安全**: `@if (user() as u)`のように絞り込んだ値の型が、ブロック内で正しく扱われます。

これらの理由から、モダンAngularでは組み込み制御フローが標準になりました。`CommonModule`への依存が消えたことは、Standalone Componentとの相性という点でも大きな意味を持ちます。

## 既存コードの移行

`*ngIf`・`*ngFor`で書かれた既存プロジェクトは、公式の移行ツールで組み込み制御フローへ自動変換できます。

```bash
ng generate @angular/core:control-flow
```

このコマンドは、テンプレート内の`*ngIf`・`*ngFor`・`*ngSwitch`を、対応する`@if`・`@for`・`@switch`へ書き換えます。`trackBy`関数も`track`式へ移し替えられます。大量のテンプレートを一括で変換できるため、移行の手間を大きく減らせます。

:::message
旧来の`*ngIf`・`*ngFor`は、現在も動作します。ただちに書き換える必要はありませんが、新規に書くコードは組み込み制御フローを使い、既存分は移行ツールで段階的に置き換えるのがよいでしょう。
:::

## 制御フローを組み合わせる

`@if`と`@for`は、入れ子にして組み合わせられます。実際の画面では、「一覧を並べつつ、条件に応じて各項目の表示を変える」といった場面が多く、その多くはこの組み合わせで表現できます。

```html
@for (task of tasks(); track task.id) {
  <div class="task">
    <span>{{ task.title }}</span>
    @if (task.done) {
      <span class="badge">完了</span>
    } @else {
      <button (click)="complete(task)">完了にする</button>
    }
  </div>
} @empty {
  <p>タスクはありません</p>
}
```

`@for`でタスクを並べ、その中の`@if`で完了状態に応じた表示を切り替えています。旧来の`*ngFor`と`*ngIf`は、同じ要素に一緒に付けられないという制約がありました。組み込み制御フローにはその制約がなく、素直に入れ子で書けます。

## よくあるつまずき

制御フローでつまずきやすい点を挙げておきます。

- **`track`の指定を忘れる**: `@for`では`track`が必須です。省略するとコンパイルエラーになります。一意なIDがあればそれを、なければ`track $index`を指定します。
- **`track`に不適切な値を指定する**: `track $index`を使うと、要素の並び替えや挿入が起きたときに、Angularが要素を取り違えて再描画のむだが生じることがあります。可能なら`track item.id`のように、要素を一意に識別できる値を選びます。
- **`@`をテキストとして書きたい**: メールアドレスなど、テンプレートに`@`を文字として出したい場合は、制御フローと誤解されないよう`&#64;`と書くか、補間`{{ '@' }}`を使います。

## まとめ

- 条件分岐は`@if`・`@else if`・`@else`、繰り返しは`@for`、値による分岐は`@switch`で書きます
- `@for`は`track`の指定が必須で、`@empty`で空のときの表示、コンテキスト変数で位置などを扱えます
- 旧来の`*ngIf`・`*ngFor`は`CommonModule`が必要でしたが、組み込み構文はimportなしで使えます
- 既存コードは`ng generate @angular/core:control-flow`で自動移行できます
- **新規開発では組み込み制御フロー（`@if`・`@for`・`@switch`）を使うのが現在の標準です**

次章では、これまで暗黙に使ってきたDirectiveそのものに目を向け、その全体像と種類を整理します。
