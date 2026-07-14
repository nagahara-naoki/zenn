---
title: "第46章 HttpClientからresource()・httpResource()へ"
---

前章で、HttpClientによる通信を学びました。通信そのものは`get`や`post`で行えますが、実際のアプリケーションでは、それだけでは足りません。「読み込み中」の表示、エラー時の表示、そして「入力が変わったら再取得する」といった処理を、自分で組み立てる必要があります。これらを手作業で書くと、意外に煩雑です。

この課題に応えるのが、Angularの新しいデータ取得のAPI、`resource()`と`httpResource()`です。これらは、Signalを土台に、非同期のデータ取得と、それにまつわる状態（読み込み中・エラー・結果）を、ひとまとめに扱います。この章では、従来のHttpClientによる書き方と比較しながら、これらの新しいAPIの考え方を学びます。

:::message
**この章で学ぶこと**

- 手作業での状態管理の課題
- `httpResource()`によるデータ取得
- Signalに反応する再取得
- 従来の書き方との違いと使い分け
:::

## 手作業での状態管理の課題

まず、従来のHttpClientで、実用的なデータ取得を書くと、どうなるかを見ます。「読み込み中」と「エラー」の状態を、自分で管理する必要があります。

```ts:従来の書き方（状態を手作業で管理）
export class UserProfile {
  private readonly http = inject(HttpClient);
  protected readonly user = signal<User | null>(null);
  protected readonly loading = signal(false);
  protected readonly error = signal<string | null>(null);

  load(id: string): void {
    this.loading.set(true);
    this.error.set(null);
    this.http.get<User>(`/api/users/${id}`).subscribe({
      next: (u) => { this.user.set(u); this.loading.set(false); },
      error: () => { this.error.set('取得に失敗しました'); this.loading.set(false); },
    });
  }
}
```

`user`・`loading`・`error`という3つの状態を、自分で用意し、通信の各段階で手作業で更新しています。成功したら`loading`を下げて`user`を設定し、失敗したら`error`を設定して`loading`を下げる。この定型的な処理を、通信のたびに書くのは、繰り返しが多く、間違いも起きやすいものです。`loading`を下げ忘れる、といったミスもよく起こります。

## httpResource()によるデータ取得

`httpResource()`は、この定型処理を、ひとまとめにします。通信と、それにまつわる状態を、ひとつのオブジェクトとして提供します。

```ts:src/app/user-profile.ts
import { Component, inject, input } from '@angular/core';
import { httpResource } from '@angular/common/http';

@Component({
  selector: 'app-user-profile',
  template: `
    @if (userResource.hasValue()) {
      <h1>{{ userResource.value().name }}</h1>
    } @else if (userResource.isLoading()) {
      <p>読み込み中</p>
    } @else if (userResource.error()) {
      <p>取得に失敗しました</p>
    }
  `,
})
export class UserProfile {
  readonly id = input.required<string>();

  // URLを返す関数を渡す。状態はすべてSignalで提供される
  protected readonly userResource = httpResource<User>(
    () => `/api/users/${this.id()}`,
  );
}
```

`httpResource(() => URL)`は、URLを返す関数を受け取ります。そして、通信の状態を、Signalとして提供します。

- `value()`: 取得したデータ
- `isLoading()`: 読み込み中かどうか
- `error()`: エラー情報
- `hasValue()`: 値があるかの安全な確認

先ほど手作業で管理していた3つの状態が、すべて`httpResource()`の中に用意されています。自分で`loading`を上げ下げする必要も、`subscribe`する必要もありません。テンプレートは、これらのSignalを読むだけで、読み込み中・エラー・成功の表示を出し分けられます。

## Signalに反応する再取得

`httpResource()`のもうひとつの強みが、Signalへの反応です。URLを返す関数の中でSignalを読んでいると、そのSignalが変わるたびに、自動で再取得が走ります。

先ほどの例では、URLの中で`this.id()`というSignal（ルートパラメーターの入力）を読んでいました。そのため、`id`が変わると、`httpResource()`は自動的に新しいURLで再取得します。第42章で`toObservable()`と`switchMap`を組み合わせて実現した「URLの変化に応じた再取得」が、`httpResource()`では、URLの関数を書くだけで実現するのです。古い通信の打ち切りも、内部で適切に扱われます。

この宣言的な書き味は、`computed()`に似ています。`computed()`が「依存するSignalから値を導く」のに対し、`httpResource()`は「依存するSignalから非同期のデータを導く」といえます。Signalの考え方が、非同期のデータ取得にまで拡張されたものだと捉えると、理解しやすくなります。

第42章では、URLの変化に応じた再取得を、`toObservable()`・`switchMap`・`toSignal()`という3つの道具を組み合わせて実現しました。`httpResource()`は、その一連を、URLを返す関数ひとつに凝縮したものだと見ることもできます。RxJSの知識がなくても、「依存するSignalを読むURL関数を書けば、変化に応じて取得し直される」という直感的な形で、リアクティブなデータ取得が書けます。もちろん、その裏側ではRxJSと同等の制御が働いており、両者は対立するものではありません。単純な取得は`httpResource()`で簡潔に、複雑な制御はRxJSで、という住み分けです。

## resource()との関係

`httpResource()`は、より汎用的な`resource()`という仕組みの、HTTP専用版です。`resource()`は、HTTPに限らず、あらゆる非同期処理（Promiseを返す処理など）を、Signalベースの状態として扱えます。

```ts:resource()の例（HTTP以外の非同期にも使える）
import { resource } from '@angular/core';

userResource = resource({
  params: () => ({ id: this.id() }),
  loader: ({ params }) => fetchUser(params.id), // Promiseを返す処理
});
```

`resource()`は`loader`にPromiseを返す関数を渡し、`httpResource()`はHTTP通信に特化してURLを渡す、という違いがあります。つまり`httpResource()`は、`resource()`のうちHTTP通信という頻出のケースを、より書きやすくしたものです。HTTP通信なら`httpResource()`が簡潔で、それ以外の非同期には`resource()`を使う、と考えるとよいでしょう。RxJSベースで書きたい場合は、第41章で触れた`rxResource()`もあります。

## 新旧の位置づけと使い分け

これらのAPIの登場時期を整理します。`resource()`はAngular 19（2024年）で、`httpResource()`はAngular 19.2（2025年）で、いずれも実験的に導入されました。その後、フィードバックを経て、v22世代で実用段階へと成熟してきています。比較的新しいAPIであるため、既存のプロジェクトの多くは、まだ従来のHttpClientを直接使っています。

使い分けの指針は、次のとおりです。

- **画面表示のためのデータ取得**: `httpResource()`が向きます。読み込み中・エラーの状態が自動で得られ、Signalへの反応で再取得も宣言的に書けます。
- **データの送信（POST・PUT・DELETE）**: これらは`httpResource()`ではなく、従来どおりHttpClientのメソッドを使います。`httpResource()`は、主に取得（GET）のための仕組みです。
- **複雑な非同期の制御**: `debounceTime`や込み入った合成が必要なら、RxJSと`toSignal()`の組み合わせが依然として有力です。

## よくあるつまずき

- **`httpResource()`で送信しようとする**: `httpResource()`は主にデータ取得（GET）のための仕組みです。データの作成・更新・削除は、従来どおりHttpClientの`post`・`put`・`delete`を使います。
- **URLの関数内でSignalを読まない**: 再取得は、URLを返す関数の中でSignalを読むことで働きます。Signalを外で固定値として展開してしまうと、変化に反応しなくなります。
- **`value()`をそのまま読む**: 値がまだ来ていない段階で`value()`を読むと、期待した値が得られないことがあります。`hasValue()`で確認してから読むのが安全です。
- **実験的な段階を軽視する**: これらのAPIは比較的新しく、細かな仕様が変わる可能性があります。採用時は、使っているAngularのバージョンでのAPIの状態を確認します。

## まとめ

- 従来のHttpClientでは、読み込み中・エラーの状態を手作業で管理する必要がありました
- `httpResource()`は、通信と状態（`value`・`isLoading`・`error`）をSignalでひとまとめにします
- URLの関数内でSignalを読むと、その変化に応じて自動で再取得されます
- `resource()`はHTTP以外の非同期にも使える汎用版で、`httpResource()`はそのHTTP専用版です
- `resource()`・`httpResource()`は、状態管理をSignalに寄せるモダンAngularの流れに沿った仕組みです
- **画面表示のためのデータ取得には`httpResource()`が有力です。送信は従来どおりHttpClientを使い、複雑な制御はRxJSと併用します**

次章では、通信にまつわる横断的な処理、Interceptorや認証、エラー・ローディングの設計を学びます。
