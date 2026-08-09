---
title: "inject()とProvider・Injectorの階層"
---

この章では、DIを実際に扱う書き方と仕組みを学びます。依存を受け取る`inject()`と、Serviceがどこで作られ共有されるのかを決めるProvider・Injectorの階層を扱います。

:::message
**この章で学ぶこと**

- Constructor Injectionによる依存の受け取り
- `inject()`関数による依存の受け取り
- Injectorの階層構造
- Serviceを提供する場所による違い
:::

## Constructor Injectionからinject()へ

前章で、依存を注入してもらうというDIの考え方を学びました。この節では、その注入を実際に受け取る書き方に踏み込みます。Angularには、依存を受け取る方法が2つあります。ひとつは、コンストラクターの引数で受け取るConstructor Injection。もうひとつが、現在の標準である`inject()`関数です。

`inject()`は、Angular 14（2022年）で導入されました。それ以前はConstructor Injectionが唯一の方法で、既存コードの多くはこの形で書かれています。両者は同じDIの仕組みを使っており、結果は変わりません。しかし、書き味と柔軟性に違いがあります。この節では、両方を比較しながら、なぜ現在は`inject()`が推奨されるのかを理解します。

### Constructor Injectionという書き方

まず、旧来のConstructor Injectionを見ます。依存を、コンストラクターの引数として宣言する方法です。

```ts:旧来の書き方（Constructor Injection）
import { Component } from '@angular/core';
import { ProductService } from './product';

@Component({ selector: 'app-product-list-page', template: `...` })
export class ProductListPage {
  constructor(private readonly service: ProductService) {}

  load(): void {
    this.service.getProducts();
  }
}
```

コンストラクターの引数に`private readonly service: ProductService`と書くと、Angularが`ProductService`を注入し、`this.service`として使えるようにします。引数に`private`などの修飾子を付けることで、そのままクラスのプロパティになる、というTypeScriptの機能を利用しています。

この書き方は長く標準でしたが、いくつかの不便がありました。依存が増えるとコンストラクターの引数が長くなること、クラスを継承したときに親のコンストラクター引数をすべて引き継いで書き直す必要があること、そして関数の中では使えないことです。

### inject()という書き方

同じ依存の受け取りを、`inject()`で書くと、次のようになります。

```ts:src/app/product-list-page.ts
import { Component, inject } from '@angular/core';
import { ProductService } from './product';

@Component({ selector: 'app-product-list-page', template: `...` })
export class ProductListPage {
  private readonly service = inject(ProductService);

  load(): void {
    this.service.getProducts();
  }
}
```

`inject(ProductService)`を、フィールドの初期化子として書きます。コンストラクターは不要です。受け取った依存は、ふつうのフィールドとして`this.service`で使えます。前章で触れたように、`inject()`は注入コンテキスト、すなわちクラスの初期化のタイミングで呼び出します。

見た目の違いはわずかですが、この違いが、いくつもの利点につながります。

### inject()の利点

`inject()`が現在推奨される理由は、Constructor Injectionの不便を解消するからです。

**コンストラクターが不要になる**。依存を受け取るためだけにコンストラクターを書く必要がなくなり、フィールドの宣言として自然に並べられます。依存が増えても、引数リストが長大になることはありません。

**継承が楽になる**。クラスを継承するとき、Constructor Injectionでは、親が受け取っている依存を子のコンストラクターでも書き、`super()`へ渡す必要がありました。`inject()`なら、親クラスの中で`inject()`しているため、子はコンストラクターに手を加えなくて済みます。

**関数の中でも使える**。これが特に大きな違いです。`inject()`は、クラスだけでなく、Angularが提供する関数の中でも呼べます。[『ルーティング応用』の章](./15-router-advanced)で学ぶ関数型のRoute Guardや、[『HTTP通信』の章](./19-http)で学ぶ関数型のInterceptorは、この性質の上に成り立っています。たとえば、Guardを次のように関数として書き、その中で依存を注入できます。

```ts:src/app/auth-guard.ts
import { inject } from '@angular/core';
import { CanActivateFn } from '@angular/router';
import { AuthService } from './auth';

export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService); // 関数の中で注入できる
  return auth.isLoggedIn();
};
```

Constructor Injectionは、コンストラクターを持つクラスでしか使えないため、こうした関数の中では依存を受け取れませんでした。`inject()`は、DIをクラスの外へ解き放ち、より柔軟な書き方を可能にしたのです。

**再利用しやすい**。複数のComponentで共通する注入と初期化の処理を、ひとつの関数にまとめて切り出せます。たとえば、`inject()`を内部で使うヘルパー関数を作り、各Componentのフィールド初期化子から呼ぶ、といった書き方ができます。これはConstructor Injectionでは難しいことでした。

### 新旧を並べて比べる

同じ依存を受け取るコードを、両者で並べます。

```ts:旧来の書き方（Constructor Injection）
export class UserPage {
  constructor(
    private readonly userService: UserService,
    private readonly logger: Logger,
  ) {}
}
```

```ts:src/app/user-page.ts（現在の書き方）
export class UserPage {
  private readonly userService = inject(UserService);
  private readonly logger = inject(Logger);
}
```

依存が増えたときの見やすさの差が分かります。`inject()`版は、各依存が独立した一行として並び、追加や削除も一行の操作で済みます。コンストラクターの引数リストを編集する必要がありません。

型の面でも`inject()`は素直です。Constructor Injectionは、引数の型注釈をAngularが読み取ってトークンを決める仕組みで、この解決はTypeScriptのコンパイル時の情報に依存していました。`inject(ProductService)`は、トークンを値として直接渡すため、何を注入しようとしているのかがコードから明快に読み取れます。戻り値の型も、渡したトークンから自動で決まるため、`this.service`の型は`ProductService`だと、型注釈を書かずとも正しく推論されます。宣言の簡潔さと型の正確さが、両立しているのです。

### 注入を細かく制御する

`inject()`には、注入のふるまいを調整するオプションを渡せます。よく使うのが、依存が見つからないときにエラーにせず`null`を許す`optional`です。

```ts
// 見つからなければ null（エラーにしない）
private readonly config = inject(AppConfig, { optional: true });
```

ほかにも、探索する範囲を制御するオプションがあります。

- **`optional`**: 見つからなければ`null`を返す（エラーにしない）
- **`self`**: 自身のInjectorだけを探す
- **`skipSelf`**: 自身を飛ばして、親のInjectorから探す
- **`host`**: 探索をホストの境界で止める

これらは、本章の次の節で学ぶInjectorの階層と関わります。ふだんは`inject(ProductService)`と書くだけで十分ですが、こうした細かな制御ができることも知っておくと、複雑な場面で役立ちます。旧来は、`@Optional()`・`@Self()`・`@SkipSelf()`といったデコレーターをコンストラクター引数に添えて、同じことを実現していました。`inject()`では、これらをオプションのオブジェクトとして渡せます。

### Constructor Injectionからinject()への移行でよくあるつまずき

- **注入コンテキストの外で`inject()`を呼ぶ**: `inject()`は、クラスの初期化時（フィールド初期化子やコンストラクター）に呼びます。ボタンクリックのハンドラーなど、初期化が終わった後の処理の中で呼ぶと、エラーになります。依存は初期化時に受け取り、フィールドに保持しておきます。
- **Constructor Injectionと混在させて混乱する**: 1つのクラスで両方を使うこともできますが、統一したほうが読みやすくなります。新規のコードは`inject()`に揃えるのがよいでしょう。
- **移行を急ぎすぎる**: 既存のConstructor Injectionは、そのままでも問題なく動きます。公式の移行スキマティクスもあるため、慌てて手作業で書き換える必要はありません。

:::message
既存プロジェクトのConstructor Injectionを`inject()`へ書き換える、公式の移行スキマティクスが用意されています。次のコマンドで実行でき、コンストラクター引数を`inject()`のフィールド宣言へ自動で変換します。

```bash
ng generate @angular/core:inject
```

差分を確認しながら段階的に進められるため、大きなプロジェクトでも安全に移行できます。
:::

## ProviderとInjectorの階層

前節までで、Serviceを定義し、それを`inject()`で受け取る流れを学びました。この節では、その裏側にもう一歩踏み込みます。注入されるインスタンスは、いったいどこで作られ、どの範囲で共有されるのでしょうか。その答えを握るのが、ProviderとInjectorの階層です。

`providedIn: 'root'`と書けば、Serviceはアプリ全体でひとつのインスタンスとして共有される、と前章で述べました。しかし、Serviceの提供のしかたは、これだけではありません。特定のComponentとその子だけで共有したい、あるいはComponentごとに別々のインスタンスを持たせたい、といった調整もできます。それを可能にするのが、Injectorの階層構造です。この節を理解すると、Serviceの「範囲」を意図して設計できるようになります。

### 2種類のInjector

Angularには、性質の異なる2つのInjectorの階層があります。

ひとつは、**EnvironmentInjector（環境インジェクター）**です。アプリケーション全体にかかわるInjectorで、`app.config.ts`の`providers`や、`providedIn: 'root'`で登録したServiceを管理します。アプリの起動時に作られ、アプリが生きている間ずっと存在する、いちばん外側のInjectorです。

もうひとつは、**ElementInjector（要素インジェクター）**です。こちらは、ComponentやDirectiveごとに作られるInjectorです。`@Component`の`providers`に登録したServiceは、このElementInjectorが管理します。Componentが生成されるときに作られ、破棄されるときに消えます。

この2つが、Componentの階層に沿って積み重なり、ひとつの階層構造をなします。頂点にEnvironmentInjectorがあり、その下に、Componentのネストに対応してElementInjectorが連なる、というイメージです。

EnvironmentInjectorは、アプリ全体の1つだけとは限りません。Routeに`providers`を指定すると、そのRoute配下に**子のEnvironmentInjector**が作られます。とくにLazy Loadingで読み込むRouteでは、その機能の中だけで使うServiceを、この場所に登録できます。

```ts:src/app/app.routes.ts
import { Routes } from '@angular/router';
import { AdminApi } from './admin/admin-api';

export const routes: Routes = [
  {
    path: 'admin',
    providers: [AdminApi], // admin配下でのみ共有される
    loadChildren: () => import('./admin/admin.routes').then((m) => m.adminRoutes),
  },
];
```

`AdminApi`は、`admin`以下のRouteでのみ有効なEnvironmentInjectorに登録されます。`providedIn: 'root'`がアプリ全体で共有されるのに対し、Routeレベルの`providers`は、その機能が表示されている間だけ生きる、中間的なスコープを作ります。ある機能でしか使わないServiceを、アプリ全体へ広げずに閉じ込めたいときに向いています。

### Serviceを提供する場所

ServiceをどのInjectorに登録するかで、そのインスタンスが共有される範囲が変わります。おもな選択肢は2つです。

ひとつは、これまで使ってきた`providedIn: 'root'`です。EnvironmentInjectorに登録され、アプリ全体でひとつのインスタンスが共有されます。ほとんどのServiceは、これで十分です。

```ts:src/app/product.ts
@Injectable({ providedIn: 'root' })
export class ProductService {}
```

もうひとつは、Componentの`providers`に登録する方法です。この場合、そのComponentのElementInjectorにServiceが登録され、Componentのインスタンスごとに別々のServiceが作られます。

```ts:src/app/editor.ts
@Component({
  selector: 'app-editor',
  providers: [DraftService], // このComponentごとにインスタンスを作る
  template: `...`,
})
export class Editor {
  private readonly draft = inject(DraftService);
}
```

`Editor`が画面に3つあれば、`DraftService`も3つ作られます。それぞれの`Editor`が、自分専用の`DraftService`を持つわけです。編集中の下書きのように、Componentごとに独立していてほしい状態を保持するのに向いています。ひとつのタブの編集内容が、別のタブに混ざらないようにしたい、といった場面が典型です。逆に、全体で共有したい状態なら`providedIn: 'root'`を選びます。提供する場所の選択が、そのまま状態の独立と共有の境界を決めるのです。

| 提供の場所 | 共有範囲 | 用途 |
|---|---|---|
| `providedIn: 'root'` | アプリ全体で単一 | 共有される状態・共通処理。大半はこちら |
| Componentの`providers` | そのComponentと子孫 | Componentごとに独立させたい状態 |

### 依存はどう解決されるか

`inject(SomeService)`が呼ばれると、Angularはどうやって該当するインスタンスを見つけるのでしょうか。答えは「下から上へ探す」です。

まず、注入を要求したComponent自身のElementInjectorを見ます。そこに登録がなければ、親のElementInjectorへ、さらにその親へと、階層を上へたどっていきます。それでも見つからなければ、最後にEnvironmentInjectorを調べます。そこにもなければ、エラーになります。

```mermaid
flowchart TD
  Env["EnvironmentInjector<br/>providedIn: 'root'"] --> P["親Componentの<br/>ElementInjector"]
  P --> C["子Componentの<br/>ElementInjector<br/>inject(Service) ここで要求"]
  C -.->|1 自身を見る| C
  C -.->|2 親へ| P
  P -.->|3 環境へ| Env
```

この「近いところから探し、なければ上へ」という仕組みが、Serviceの範囲を階層で制御できる理由です。あるComponentの`providers`に登録すれば、そのComponentと子孫は、上位の同名Serviceより先に、その登録を見つけます。近いInjectorの登録が優先されるのです。前節で触れた`self`や`skipSelf`といったオプションは、この探索の起点や範囲を調整するものでした。

たとえば`skipSelf`は、自身のElementInjectorを探索から外し、親から探し始めます。あるServiceを`providers`に登録したComponentと同じ要素にDirectiveを置くと、両者は同じElementInjectorを共有します。このDirectiveがそのServiceを注入すると、既定では同じ要素のインスタンスを受け取ります。ホスト側ではなく、ひとつ外側に登録された同名のServiceを使いたいときは、`skipSelf`で自身を飛ばします。

```ts:src/app/section-outline.ts
import { Directive, inject } from '@angular/core';
import { SectionContext } from './section-context';

@Directive({ selector: '[appSectionOutline]' })
export class SectionOutline {
  // 同じ要素のComponentが提供するSectionContextではなく、
  // ひとつ外側（親）のSectionContextを参照する
  private readonly outer = inject(SectionContext, { skipSelf: true });
}
```

`self`はこの逆で、探索を自身のElementInjectorだけに限り、親をたどりません。入れ子になった同名Serviceのうち、どの階層のものを使うかを、これらのオプションで意図して選べます。

## Providerを用途に応じて定義する

### Providerの種類

ここまで、Serviceのクラスをそのまま登録してきました。実は、`providers`への登録には、いくつかの書き方（Provider）があります。「このトークンに対して、何を、どう用意するか」を指定するものです。

- **`useClass`**: 指定したクラスのインスタンスを用意する。クラス名だけを書く通常の登録は、この省略形です。
- **`useValue`**: あらかじめ用意した値（オブジェクトや定数）を、そのまま提供する。
- **`useFactory`**: 関数を呼び、その戻り値を提供する。生成に条件や計算が必要なときに使う。
- **`useExisting`**: 既存のトークンの別名を作る。あるトークンへの要求を、別のトークンへ振り向ける。

たとえば、あるインターフェースの実装を差し替えたいときは、`useClass`が使えます。テスト時に本物のServiceを偽物に差し替える、という前章で触れた話は、この仕組みで実現します。

```ts
// 本番ではRealLoggerを、テストではFakeLoggerを注入する
providers: [{ provide: Logger, useClass: RealLogger }]
```

`provide`が要求されるトークン、`useClass`が実際に用意するクラスです。使う側は`inject(Logger)`と書くだけで、登録を`RealLogger`から`FakeLogger`に変えれば、中身が差し替わります。使う側のコードは、まったく変わりません。

生成に条件や計算が絡むときは、`useFactory`を使います。ファクトリー関数は注入コンテキストで実行されるため、関数の中で`inject()`を呼び、別のServiceや設定値を受け取りながら、返すインスタンスを組み立てられます。

```ts:src/app/logger.provider.ts
import { inject, Provider } from '@angular/core';
import { Logger } from './logger';
import { RealLogger } from './real-logger';
import { NoopLogger } from './noop-logger';
import { RuntimeFlags } from './runtime-flags';

export const loggerProvider: Provider = {
  provide: Logger,
  useFactory: () => {
    const flags = inject(RuntimeFlags); // ファクトリー内でも注入できる
    return flags.debug ? new RealLogger() : new NoopLogger();
  },
};
```

`RuntimeFlags`の`debug`に応じて、返すLoggerを切り替えています。かつては`deps`プロパティに依存を並べて渡していましたが、ファクトリー内で`inject()`が呼べるようになったため、その記述はいりません。

`useExisting`は、既存のトークンに別名を与えます。あるトークンへの要求を、登録済みの別のトークンへ振り向けるもので、1つのインスタンスを複数の名前で参照させるアダプターとして働きます。

```ts
providers: [
  FileLogger,
  { provide: Logger, useExisting: FileLogger }, // Loggerとしても同じFileLoggerを返す
]
```

`inject(Logger)`と`inject(FileLogger)`は、どちらも同一の`FileLogger`インスタンスを返します。`useClass`が新しいインスタンスを作るのに対し、`useExisting`は既存のインスタンスを共有する点が異なります。広く使えるインターフェース（`Logger`）で受け取りつつ、実体は具体的なService（`FileLogger`）にしておく、といった設計に向きます。

使い分けの目安を挙げます。素直にクラスを提供するなら`useClass`（クラス名だけの記法もこれにあたります）、できあいの値や設定オブジェクトなら`useValue`、生成に他の依存や分岐が要るなら`useFactory`、既存の登録に別名を付けたいなら`useExisting`を選びます。

### クラスでない依存とInjectionToken

依存は、いつもクラスとは限りません。設定オブジェクトや、APIのURLといった文字列など、クラスでない値を注入したいこともあります。しかし、そうした値はトークンとして使えません。`inject('https://...')`のような指定は成り立たないためです。

この問題を解くのが、`InjectionToken`です。クラスでない依存のために、一意な目印を作る仕組みです。

```ts:src/app/config.ts
import { InjectionToken } from '@angular/core';

export const API_BASE_URL = new InjectionToken<string>('API_BASE_URL');
```

```ts:src/app/app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [{ provide: API_BASE_URL, useValue: 'https://api.example.com' }],
};
```

`API_BASE_URL`というトークンを作り、`useValue`で実際のURLを登録しています。使う側は、このトークンで注入します。

```ts
private readonly baseUrl = inject(API_BASE_URL);
```

`InjectionToken`により、文字列や設定オブジェクトのような、クラスでない値も、型安全にDIの仕組みへ載せられます。名前の衝突を避けつつ、値を注入できるのです。環境ごとに異なるAPIのURLや、機能のオン・オフを切り替える設定などを、この方法でアプリ全体に配る、といった使い方が典型です。ハードコードを避け、設定を一か所に集められるため、環境の切り替えや設定変更に強い構成になります。

`InjectionToken`には、既定値を持たせることもできます。第2引数に`factory`を渡すと、`providers`への登録がなくても、そのトークン自身が既定のインスタンスを用意します。

```ts:src/app/tokens.ts
import { InjectionToken } from '@angular/core';

export const WINDOW = new InjectionToken<Window>('WINDOW', {
  providedIn: 'root',
  factory: () => window,
});
```

こう定義しておくと、どこにも登録せずに`inject(WINDOW)`がそのまま使えます。`providedIn: 'root'`と`factory`を組み合わせたトークンは、どこからも使われなければ最終的なバンドルから取り除かれる（Tree Shakingが効く）ため、既定値を持つトークンの定義方法として推奨されます。

### 同じトークンに複数登録するmulti

ここまでのProviderは、1つのトークンに1つの提供を対応させるものでした。`multi: true`を付けると、**同じトークンに複数のProviderを登録**でき、注入時にはそれらをまとめた配列を受け取ります。

```ts:src/app/app.config.ts
import { ApplicationConfig, InjectionToken } from '@angular/core';

export const VALIDATORS = new InjectionToken<Validator[]>('VALIDATORS');

export const appConfig: ApplicationConfig = {
  providers: [
    { provide: VALIDATORS, useValue: lengthValidator, multi: true },
    { provide: VALIDATORS, useValue: patternValidator, multi: true },
  ],
};
```

```ts
// 登録されたすべての値が配列で届く
private readonly validators = inject(VALIDATORS); // Validator[]
```

同じトークンに`multi: true`で登録すると、`inject()`は個々の値ではなく、それらを集めた配列を返します。この仕組みは、あとから機能を差し込める「拡張点」を作るときに効きます。Angular自身も、複数のInterceptorやアプリ起動時の初期化処理を束ねる土台として`multi`を使っています。中心のコードを書き換えるのではなく、登録を1つ足すだけで機能を追加できるのが利点です。

同じトークンに対して、`multi: true`を付けたProviderと付けないProviderを混在させることはできません。どちらかに統一します。

### ProviderとInjectorの階層でよくあるつまずき

- **どこに提供すべきか迷う**: まずは`providedIn: 'root'`を基本と考えます。Componentごとに独立したインスタンスが必要な、明確な理由があるときだけ、Componentの`providers`を使います。
- **意図せず複数インスタンスができる**: `providedIn: 'root'`のServiceを、さらにComponentの`providers`にも登録すると、そのComponent配下では別インスタンスになります。共有したいServiceを二重に登録していないか、確認します。
- **`NullInjectorError`が出る**: 「Providerが見つからない」というエラーです。Serviceに`providedIn: 'root'`が付いているか、あるいは適切なInjectorに登録されているかを確認します。エラーメッセージには、どのトークンが解決できなかったかが示されるため、その名前を手がかりに登録漏れを探します。

## まとめ

- 依存を受け取る方法には、Constructor Injectionと`inject()`があります
- `inject()`はAngular 14で導入され、フィールド初期化子として依存を受け取ります
- `inject()`は、コンストラクター不要・継承が楽・関数内でも使える、という利点があります
- **新規開発では、依存の受け取りに`inject()`を使うのが現在の標準です**
- Injectorには、アプリ全体のEnvironmentInjectorと、ComponentごとのElementInjectorがあります
- `providedIn: 'root'`はアプリ全体で単一、Componentの`providers`はComponentごとに別インスタンスです
- 依存は、要求元から上へInjectorをたどって解決され、近い登録が優先されます

次章からは、Angularが画面をいつ更新するのかという変更検知と、その中心にあるSignalsに入ります。
