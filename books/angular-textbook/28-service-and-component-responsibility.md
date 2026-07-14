---
title: "第22章 ServiceとComponentの責務"
---

第2部から第4部にかけて、Componentの作り方と、Component間のやり取りを学んできました。ここまでのアプリケーションは、いわばComponentだけで組み立てられていました。しかし、規模が大きくなると、Componentだけでは無理が生じます。データの取得、業務ルールの計算、複数のComponentで共有したい状態。こうした関心事を、画面を担うComponentに押し込めると、たちまち見通しが悪くなります。

そこで登場するのがServiceです。Serviceは、Componentから切り出した処理を受け持つ、ふつうのクラスです。この章では、Serviceとは何か、Componentとどのように責務を分ければよいのかを、具体例を通して整理します。責務の分け方は、アプリケーション全体の設計を左右する重要な判断です。

:::message
**この章で学ぶこと**

- Serviceとは何か、なぜ必要か
- `@Injectable`によるServiceの定義
- ComponentとServiceの責務の分け方
- 状態を持つServiceと持たないService
:::

## Serviceとは何か

Serviceは、特定の役割を持った処理をまとめたクラスです。画面を持たない点が、Componentとの大きな違いです。Componentが「見た目と操作」を担うのに対し、Serviceは「その裏で動く処理」を担います。たとえば、次のような処理がServiceの担当です。

- サーバーとのデータのやり取り（API通信）
- 業務上の計算やルールの適用
- 複数のComponentで共有したいデータの保持
- ログの記録や、外部ライブラリの呼び出し

これらをComponentから切り離すと、Componentは「受け取って表示する」「操作を受け付ける」という本来の役割に集中できます。第10章で「1つのComponentは1つの関心事に集中する」と述べましたが、Serviceは、その関心事の分離を、画面の外側にまで広げる手段だといえます。

## なぜComponentから切り出すのか

処理をServiceへ切り出す理由は、大きく3つあります。

1つ目は、**再利用**です。たとえば「商品データを取得する処理」を`ProductService`に置けば、一覧画面でも詳細画面でも、同じServiceを使えます。もしこの処理を一覧Componentの中に書いてしまうと、詳細画面で使うために、同じコードをもう一度書くことになります。

2つ目は、**見通し**です。Componentがデータ取得も計算も抱えていると、テンプレートとの対応関係が読み取りにくくなります。処理をServiceへ追い出せば、Componentのクラスは「Serviceを呼び、結果を表示に橋渡しする」だけの、薄く読みやすいものになります。

3つ目は、**テストのしやすさ**です。業務ロジックがServiceに分かれていれば、画面を通さずに、そのロジック単体をテストできます。Componentのテストと、ロジックのテストを、それぞれの関心に応じて分けられます。テストについては第11部で詳しく扱いますが、責務の分離は、テスト容易性の土台になります。

## @InjectableでServiceを定義する

Serviceは、`@Injectable`デコレーターを付けたクラスとして定義します。Angular CLIで生成できます。

```bash
ng generate service product
```

生成される基本形は、次のようになります。

```ts:src/app/product.ts
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root',
})
export class ProductService {
  getProducts(): Product[] {
    // データを返す処理（詳細は第9部のHTTP通信で扱う）
    return [/* 省略 */];
  }
}
```

注目すべきは`@Injectable({ providedIn: 'root' })`です。`providedIn: 'root'`は、「このServiceをアプリケーション全体で使えるように登録する」という指定です。この一行があると、Serviceはアプリのどこからでも受け取れるようになります。しかも、実際に使われている場合にだけ最終的な成果物に含まれるため、使わないServiceは自動的に取り除かれます。この仕組みは、次章以降で詳しく扱うDIの一部です。

## Serviceはひとつのインスタンスが共有される

`providedIn: 'root'`で登録したServiceは、アプリケーション全体でひとつのインスタンスだけが作られ、それがすべての利用者で共有されます。あるComponentが受け取る`ProductService`も、別のComponentが受け取る`ProductService`も、同じひとつのインスタンスです。

この性質は、状態を共有したいときに役立ちます。たとえば、`ProductService`が取得済みの商品データを保持していれば、複数の画面がその同じデータを参照できます。第17章で触れた、親子を越えた状態共有の課題を、Serviceが解決する糸口がここにあります。共有される単一のインスタンスを通じて、離れたComponentどうしが同じ状態を見られるのです。

一方で、共有されるということは、状態の扱いに注意が要るということでもあります。あるComponentがServiceの状態を変えれば、その変化はほかのすべての利用者に及びます。この点は、状態管理を扱う第10部で、あらためて掘り下げます。

## ComponentとServiceの責務を分ける

では、具体的に何をComponentに残し、何をServiceへ切り出せばよいのでしょうか。目安を整理します。

| 担当 | Componentに残すもの | Serviceへ切り出すもの |
|---|---|---|
| 画面 | テンプレートとの橋渡し、表示用の状態 | — |
| データ | 表示に必要な値の保持 | データの取得・保存（API通信） |
| ロジック | 表示の出し分けなど画面固有の判断 | 業務ルール、共有される計算 |
| 状態 | そのComponentだけの一時的な状態 | 複数のComponentで共有する状態 |

判断に迷ったときの問いは、「この処理は、この画面だけのものか」です。その画面に固有のことならComponentに、ほかでも使う、あるいは画面と切り離せることならServiceに置く、と考えます。次のコードは、Componentが`ProductService`に取得を任せ、自分は表示への橋渡しに徹する例です。

```ts:src/app/product-list-page.ts
@Component({
  selector: 'app-product-list-page',
  template: `
    @for (p of products(); track p.id) {
      <p>{{ p.name }}</p>
    }
  `,
})
export class ProductListPage {
  private readonly service = inject(ProductService);
  protected readonly products = signal(this.service.getProducts());
}
```

Componentのクラスは、Serviceを受け取り、その結果を`products`として表示に渡すだけです。データをどこから、どうやって取ってくるかは、`ProductService`の関心事であり、Componentは知りません。この分担が、両者の見通しを保ちます。

## 状態を持つServiceと持たないService

Serviceは、大きく2種類に分けて考えると整理しやすくなります。

ひとつは、**状態を持たないService**です。渡された入力から結果を計算して返すだけで、自分ではデータを抱えません。税額の計算や、日付の整形といった、道具のようなServiceがこれにあたります。呼ぶたびに同じ入力なら同じ結果を返すため、扱いが単純で、テストも容易です。次は、価格の計算をまとめた状態を持たないServiceの例です。

```ts:src/app/pricing.ts
@Injectable({ providedIn: 'root' })
export class PricingService {
  withTax(price: number): number {
    return Math.floor(price * 1.1);
  }

  applyDiscount(price: number, rate: number): number {
    return Math.floor(price * (1 - rate));
  }
}
```

このServiceは、どのComponentから呼ばれても、渡された値だけで結果を決めます。内部に状態がないため、動きが予測しやすく、単体テストも「入力を渡して戻り値を確かめる」だけで済みます。業務ルールが各Componentに散らばるのを防ぎ、計算の根拠を一か所に集められる点も利点です。

もうひとつは、**状態を持つService**です。取得したデータや、アプリの現在の状態を、自身の中に保持します。共有される単一インスタンスという性質を活かし、複数のComponentから参照される「状態の置き場所」として働きます。第10部で学ぶStore Serviceは、この発展形です。

どちらのServiceも役割がありますが、状態を持つServiceは、変化の伝わり方に注意が必要です。モダンAngularでは、Serviceが持つ状態をSignalで表現することで、その変化をComponentへ自然に伝えられます。Serviceの中で`signal()`を使って状態を保持し、Componentはそれを読み取るだけで、変化に追従した表示が得られるのです。この手法は、第6部のSignalsと、第10部の状態管理で詳しく扱います。まずは「状態を持つか持たないか」でServiceを見分ける視点を持っておくと、設計の判断がしやすくなります。

## よくあるつまずき

- **Componentに何でも書いてしまう**: 動くものを速く作ろうとすると、つい取得も計算もComponentに書きがちです。重複や肥大化が見えたら、Serviceへの切り出しを検討します。
- **Serviceを細かく分けすぎる**: 逆に、意味のまとまりがないのにServiceを乱立させると、全体像が追いにくくなります。関連する処理は、ひとつのServiceにまとめます。
- **`providedIn: 'root'`を書き忘れる**: これがないと、Serviceをどこで使えるようにするかを別途指定する必要があります。まずは`root`を基本と考えておくと迷いません。

## まとめ

- Serviceは、Componentから切り出した処理を受け持つ、画面を持たないクラスです
- データ取得・業務ロジック・共有状態などをServiceへ移すと、Componentが薄く保てます
- `@Injectable({ providedIn: 'root' })`で、アプリ全体で使えるServiceを定義します
- `providedIn: 'root'`のServiceは単一インスタンスが共有され、状態の共有にも使えます
- 「この画面だけのものか」を基準に、ComponentとServiceの責務を分けます

次章では、作ったServiceをComponentが受け取る仕組み、Dependency Injectionそのものを学びます。
