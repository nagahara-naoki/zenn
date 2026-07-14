---
title: "第16章 Pipeとテンプレートの再利用"
---

第3部もこの章で最後です。ここまで、データバインディング・制御フロー・Directiveと、テンプレートを動かす道具を学んできました。最後に扱うのがPipe（パイプ）です。Pipeは、テンプレートの中で値を整形するための仕組みです。

日付を「2026年7月14日」の形で表示したい、数値を通貨の形式にしたい、文字列を大文字にしたい。こうした「表示のための変換」を、クラスを書き換えずにテンプレート側で完結できるのがPipeです。Angularには便利な組み込みPipeが多数用意されており、独自のPipeを作ることもできます。この章で、その使い方と仕組みを押さえます。

:::message
**この章で学ぶこと**

- Pipeによる値の整形
- 代表的な組み込みPipe
- 自作Pipeの作り方
- Pure PipeとImpure Pipe、そしてAsyncPipe
:::

## Pipeとは

Pipeは、テンプレート内で値を変換する仕組みです。縦棒`|`の記号を使い、「値 | Pipe名」の形で書きます。この記号が、Unixのパイプのように「左の値を右へ流して加工する」イメージを表しています。

```html
<p>{{ name() | uppercase }}</p>
```

`name()`が`'yamada'`なら、`uppercase`というPipeを通って`'YAMADA'`と表示されます。元の`name`の値は変わりません。Pipeは、あくまで表示のための変換を行うだけです。クラス側のデータはそのままに、見せ方だけを整えられるのが特徴です。

## 代表的な組み込みPipe

Angularには、よく使う変換があらかじめPipeとして用意されています。代表的なものを挙げます。

| Pipe | 用途 | 例 |
|---|---|---|
| `date` | 日付の書式化 | `{{ today \| date }}` |
| `currency` | 通貨表示 | `{{ price \| currency }}` |
| `number` | 数値の桁区切り | `{{ count \| number }}` |
| `percent` | パーセント表示 | `{{ rate \| percent }}` |
| `uppercase` / `lowercase` | 大文字・小文字化 | `{{ name \| uppercase }}` |
| `json` | オブジェクトのJSON表示 | `{{ data \| json }}` |
| `async` | Observable・Promiseの購読 | `{{ data$ \| async }}` |

`json`は、デバッグ中にオブジェクトの中身を確認したいときに重宝します。`date`や`currency`は、地域（ロケール）に応じた書式で表示してくれます。これらの組み込みPipeは、いずれもimportなしでテンプレートに書けます。

## パラメーターとチェイン

Pipeには、コロン`:`でパラメーターを渡せます。たとえば`date`Pipeは、書式の文字列を受け取ります。

```html
<p>{{ today() | date:'yyyy年MM月dd日' }}</p>
```

これで「2026年07月14日」のような表示になります。パラメーターを複数渡すときは、コロンを重ねます。

```html
<p>{{ price() | currency:'JPY':'symbol' }}</p>
```

さらに、Pipeは連結（チェイン）できます。左から右へ順に適用されます。

```html
<p>{{ today() | date:'fullDate' | uppercase }}</p>
```

この例では、まず`date`で日付を書式化し、その結果を`uppercase`で大文字にしています。複数の変換を、テンプレート上で簡潔に組み合わせられます。

## 自作Pipeを作る

組み込みPipeにない変換は、自分で作れます。`@Pipe`デコレーターを付け、`PipeTransform`インターフェースの`transform`メソッドを実装します。Angular CLIで生成できます。

```bash
ng generate pipe truncate
```

例として、長い文字列を指定の長さで切り詰める`truncate`Pipeを作ります。

```ts:src/app/truncate-pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'truncate',
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit = 20, suffix = '...'): string {
    if (value.length <= limit) {
      return value;
    }
    return value.slice(0, limit) + suffix;
  }
}
```

`transform`の第1引数が、パイプの左側から流れてくる値です。第2引数以降が、コロンで渡すパラメーターに対応します。使うときは、Componentの`imports`に`TruncatePipe`を宣言したうえで、次のように書きます。

```html
<p>{{ description() | truncate:30 }}</p>
```

`description()`が30文字を超えると、30文字＋`...`に切り詰められます。パラメーターに既定値を持たせているため、`| truncate`と引数なしでも動きます。

なお、Pipeも第7章で学んだStandaloneの考え方が適用され、Angular 19以降は`standalone: true`の指定が不要です。単体でimportして使えます。

## Pure PipeとImpure Pipe

Pipeには、Pure（純粋）とImpure（非純粋）の2種類があります。既定はPureです。

Pure Pipeは、入力値が変わったときにだけ`transform`を実行します。正確には、入力の参照が変わったときです。同じ入力に対しては結果をそのまま使い回すため、効率的です。組み込みPipeの多くはPureで、通常はこれで十分です。

一方、配列やオブジェクトの「中身」が変わっても、参照が同じなら、Pure Pipeは再実行されません。中身の変化に追従したい場合は、`pure: false`を指定してImpure Pipeにします。

```ts:src/app/filter-pipe.ts
@Pipe({
  name: 'filterActive',
  pure: false,
})
export class FilterActivePipe implements PipeTransform {
  transform(items: Item[]): Item[] {
    return items.filter((item) => item.active);
  }
}
```

ただし、Impure Pipeは変更検知のたびに実行されるため、パフォーマンスに影響します。多用は避け、まずはデータ側を新しい参照として更新する（配列を作り直す）ことで、Pure Pipeのまま対応できないかを検討するのが望ましい書き方です。

## AsyncPipeで非同期の値を扱う

組み込みPipeの中でも特に重要なのが、`async`（AsyncPipe）です。これは、ObservableやPromiseから値を取り出し、テンプレートに表示します。

```html
<p>{{ user$ | async }}</p>
```

`user$`がObservableのとき、`async`Pipeは自動的にそれを購読（subscribe）し、流れてきた値を表示します。さらに、Componentが破棄されるときには購読を自動で解除してくれます。手動での購読と解除が不要になるため、メモリリークを防ぎつつ簡潔に書けます。

実務では、`async`Pipeを`@if`と組み合わせ、値が届いてから表示する書き方がよく使われます。第12章で学んだ`as`で、購読した値に名前を付けられます。

```html
@if (user$ | async; as user) {
  <p>{{ user.name }}</p>
} @else {
  <p>読み込み中</p>
}
```

こうすると、`user`が届くまでは「読み込み中」を表示し、届いたら中身を表示できます。`user$ | async`を何度も書かずに、`user`という名前で使い回せる点も利点です。ObservableやRxJSの詳しい仕組みは第8部で扱いますが、その値をテンプレートに映す入り口が、この`async`Pipeだと覚えておいてください。

## Pipeとメソッド呼び出しの違い

同じ変換は、クラスのメソッドを補間で呼んでも実現できます。それでもPipeを使う利点は、主に2つあります。

- **再利用性**: Pipeは、一度作れば複数のComponentで使い回せます。メソッドは、そのクラスの中に閉じてしまいます。
- **効率**: Pure Pipeは入力が変わったときだけ再計算します。テンプレートでメソッドを呼ぶと、変更検知のたびに実行される可能性があり、むだが生じやすくなります。

表示のための変換で、複数の場所で使う可能性があるものは、Pipeに切り出すのが適切です。逆に、そのComponentだけで一度きり使う単純な変換なら、`computed()`やメソッドでも構いません。

## Signalとの関係

モダンAngularでは、状態をSignalで持つことが増えています。Signalから導いた値の整形は、Pipeで行うことも、`computed()`で行うこともできます。使い分けの目安は、その変換が「表示のためのものか」「複数箇所で再利用するか」です。

`uppercase`や`date`のような汎用的な表示変換はPipeが向きます。一方、そのComponent固有の、状態から別の状態を導く計算は、`computed()`のほうが自然です。両者は競合するものではなく、役割に応じて併用します。Signalと`computed()`の詳細は、第6部で改めて扱います。

## よくあるつまずき

Pipeでつまずきやすい点を挙げておきます。

- **`imports`への宣言忘れ**: 自作Pipeや、一部の組み込みPipeは、使う側のComponentの`imports`に宣言が必要です。`{{ value | truncate }}`が効かないときは、まずここを確認します。
- **Impure Pipeの多用**: `pure: false`のPipeは変更検知のたびに実行され、パフォーマンスを損ないます。配列の絞り込みなどは、Pipeではなくデータ側（`computed()`など）で行うほうが安全な場面が多くあります。
- **Pipeで副作用を起こす**: `transform`は、値を変換して返すことに徹するべきです。中でデータを書き換えたり通信したりすると、予期しない再実行で問題が起きます。Pipeは純粋な変換だけを担う、と考えてください。

## まとめ

- Pipeは、テンプレート内で値を整形する仕組みで、縦棒`|`で書きます
- `date`・`currency`・`uppercase`・`json`・`async`など、便利な組み込みPipeが用意されています
- パラメーターはコロン`:`で渡し、複数のPipeは連結できます
- 自作Pipeは`@Pipe`と`transform`メソッドで作り、既定のPure Pipeが効率的です
- `async`PipeはObservableを自動で購読・解除し、非同期の値を安全に表示します

以上で第3部は終わりです。次の第4部では、複数のComponentの間で状態をやり取りする方法、すなわち`input()`や`output()`によるデータの受け渡しを学びます。
