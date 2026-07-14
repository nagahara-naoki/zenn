---
title: "第19章 @Outputからoutput()へ"
---

前章では、親から子へデータを渡す入力を学びました。この章では、その逆、子から親へ出来事を伝える「出力」を扱います。ボタンが押された、項目が選ばれた、といった子の中で起きたことを、親に知らせる仕組みです。

現在の標準は、`output()`関数です。Angular 17.3（2024年）で導入されました。旧来は`@Output`デコレーターと`EventEmitter`を組み合わせて書いていました。この章では、`output()`の書き方を、旧来の方式と比較しながら学びます。入力の`input()`と対になる仕組みなので、あわせて理解すると、親子間のやり取りの全体像がつかめます。

:::message
**この章で学ぶこと**

- `output()`による出力の宣言
- `emit()`による値の送出
- 別名の指定と、出力の命名
- 旧来の`@Output`・`EventEmitter`との違い
:::

## output()で出力を宣言する

子から親へ出来事を伝えるには、`output()`で出力を宣言します。

```ts:src/app/like-button.ts
import { Component, output } from '@angular/core';

@Component({
  selector: 'app-like-button',
  template: `<button (click)="liked.emit()">いいね</button>`,
})
export class LikeButton {
  liked = output<void>();
}
```

`liked = output<void>()`で、`liked`という出力を定義しています。型引数の`<void>`は、この出力が値を伴わない、ただの通知であることを表します。ボタンがクリックされたら、`liked.emit()`で出来事を送り出します。親は、イベントバインディングで受け取ります。

```html
<app-like-button (liked)="onLiked()" />
```

子の`liked.emit()`が呼ばれると、親の`onLiked()`が実行されます。第3部で学んだイベントバインディング`(event)`が、標準のDOMイベントだけでなく、こうした自作の出力にも使える、というわけです。

## 値を伴う出力

出力には、値を添えて送ることもできます。型引数に、送りたい値の型を指定します。

```ts:src/app/rating.ts
@Component({
  selector: 'app-rating',
  template: `
    @for (star of stars; track star) {
      <button (click)="rated.emit(star)">★</button>
    }
  `,
})
export class Rating {
  protected readonly stars = [1, 2, 3, 4, 5];
  rated = output<number>();
}
```

`rated = output<number>()`は、数値を伴う出力です。星がクリックされると、その数を`rated.emit(star)`で送ります。親側では、`$event`でその値を受け取れます。

```html
<app-rating (rated)="onRated($event)" />
```

```ts
protected onRated(value: number): void {
  console.log(`${value}が選ばれました`);
}
```

`$event`には、`emit`に渡した値がそのまま入ります。オブジェクトを送ることもでき、複数の情報をまとめて伝えられます。

実践的な例として、検索ボックスを考えます。入力された語を、確定のタイミングで親に伝える部品です。子は「検索語が確定した」という出来事だけを送り、その語で実際に何を検索するかは親が決めます。

```ts:src/app/search-box.ts
@Component({
  selector: 'app-search-box',
  template: `
    <input #box (keyup.enter)="search.emit(box.value)" />
    <button (click)="search.emit(box.value)">検索</button>
  `,
})
export class SearchBox {
  search = output<string>();
}
```

```html
<!-- 親：検索語を受け取って一覧を絞り込む -->
<app-search-box (search)="applyFilter($event)" />
```

`SearchBox`は、検索の実行方法を何も知りません。語を送り出すことに徹しているため、商品一覧でもユーザー一覧でも、同じ部品を使い回せます。これが、第17章で述べた「出力は通知に徹する」設計の具体的な効き目です。

## 別名と命名の指針

出力にも、テンプレートで使う名前を変える別名（alias）を指定できます。

```ts
changed = output<string>({ alias: 'valueChanged' });
```

命名にはいくつかの慣習があります。

- **キャメルケースで書く**: `valueChanged`のように、先頭を小文字にしたキャメルケースにします。
- **`on`を付けない**: `onLiked`のような`on`接頭辞は避けます。`on`は、受け取る親側のハンドラーに付ける習慣があるためです。出力名は、起きた出来事を表す`liked`・`saved`・`deleted`のような名前にします。
- **過去形が自然**: 出来事は「起きたこと」なので、`saved`・`closed`のような過去形がなじみます。

なお、Angularの出力は、標準のDOMイベントと違い、DOMツリーをさかのぼって伝わること（バブリング）はありません。あくまで、直接の親子の間で伝わる通知です。

## 旧来の@Outputとの比較

`output()`が登場する前は、`@Output`デコレーターと`EventEmitter`を組み合わせていました。同じ`liked`出力を、旧来の書き方で示します。

```ts:旧来の書き方（@OutputとEventEmitter）
import { Component, EventEmitter, Output } from '@angular/core';

@Component({
  selector: 'app-like-button',
  template: `<button (click)="liked.emit()">いいね</button>`,
})
export class LikeButton {
  @Output() liked = new EventEmitter<void>();
}
```

`@Output()`を付けたプロパティに、`EventEmitter`のインスタンスを代入します。`emit()`で送出する点は同じです。動作は`output()`版と変わりませんが、次のような違いがあります。

- **`EventEmitter`が不要**: `output()`は、`EventEmitter`をインスタンス化する必要がありません。`output<void>()`と関数を呼ぶだけです。
- **役割が明確**: `EventEmitter`は名前に「Emitter（送出するもの）」とありながら、内部的にはRxJSのObservableを継承しており、購読もできてしまう、あいまいな存在でした。`output()`は「出力を宣言する」という役割に絞られ、誤用しにくくなっています。
- **`input()`との一貫性**: 入力が`input()`、出力が`output()`と、関数呼び出しで揃います。宣言の形が統一され、読みやすくなります。

## ObservableをもとにするoutputFromObservable

補足として、RxJSのObservableから出力を作る`outputFromObservable()`という関数もあります。Observableが値を流すたびに、それを出力として送り出したいときに使います。RxJSについては第8部で扱うため、ここでは「Observableと出力をつなぐ手段がある」とだけ知っておけば十分です。

```ts
import { outputFromObservable } from '@angular/core/rxjs-interop';

// data$（Observable）が流す値を、そのまま出力にする
dataChanged = outputFromObservable(this.data$);
```

通常の用途では、`output()`と`emit()`で十分です。既存のObservableを出力に変換したい、という限られた場面のための道具だと捉えてください。

なお、`output()`が返すものは、親のイベントバインディングで受けるのが基本ですが、Componentを動的に生成する高度な場面では、`subscribe()`で購読することもできます。この場合は、不要になったときに購読を解除する責任が生じます。ふだんのテンプレート経由のバインディングでは、購読も解除もAngularが引き受けるため、こうした後始末を意識する必要はありません。この手軽さも、テンプレートでのやり取りを基本とすべき理由のひとつです。

## 複数の情報をまとめて送る

出来事に複数の情報が伴うときは、オブジェクトにまとめて送ると扱いやすくなります。たとえば、並べ替えの操作を「どの列を、どの向きで」という2つの情報とともに伝える場合です。

```ts:src/app/sort-header.ts
type SortEvent = { column: string; direction: 'asc' | 'desc' };

@Component({ selector: 'app-sort-header', template: `...` })
export class SortHeader {
  sortChanged = output<SortEvent>();

  protected toggle(column: string): void {
    this.sortChanged.emit({ column, direction: 'asc' });
  }
}
```

型に名前（`SortEvent`）を付けておくと、親側で`$event`を受け取るときも、その形が明確になります。引数を増やしたいときにオブジェクトを1つ渡す形にしておけば、後から情報を足しやすい、という利点もあります。1つの出力で多くを伝えたい場面では、この「オブジェクトにまとめる」手法が有効です。

## よくあるつまずき

- **出力を入力のように使おうとする**: 出力は、子から親への一方向の通知です。親から子へ値を渡したいなら、出力ではなく入力（`input()`）を使います。
- **`emit`の呼び忘れ**: 出力は宣言しただけでは何も起きません。出来事が起きたタイミングで`emit()`を呼んで、はじめて親に伝わります。
- **`on`接頭辞を出力名に付ける**: `onSave`のような名前は避け、`saved`とします。`on`は親のハンドラー側の習慣です。

## まとめ

- `output()`は、子から親へ出来事を伝える、出力の宣言です
- `emit()`で送出し、値を添えれば親は`$event`で受け取れます
- 出力名はキャメルケースで、`on`を付けず、起きた出来事を表す名前にします
- 旧来の`@Output`は`EventEmitter`を要し、購読もできるあいまいさがありました
- **新規開発では`output()`を使うのが現在の標準です。`@Output`は既存コードの理解のために押さえます**

次章では、入力と出力を1つにまとめ、双方向のやり取りを実現する`model()`を学びます。
