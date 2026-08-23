---
title: "付録B 非推奨・古い書き方からの移行"
---

RxJSは長く使われてきたライブラリで、古い書き方が残ったコードも多くあります。この付録では、既存コードを読んだり書き換えたりするときに知っておきたい、代表的な移行をまとめます。

本書はRxJS 7.8.2を基準にし、2026年8月23日に記述を確認しました。ここで挙げる書き方には、7.8でまだ動くものもあります。単に置き換えるのではなく、利用中のRxJSのバージョンを確認してから移行してください。

## toPromiseからfirstValueFrom・lastValueFromへ

`toPromise`は、RxJS 7で非推奨です。最初の値と最後の値のどちらを待つのか、値なしの完了をどう扱うのかが曖昧になるため、`firstValueFrom`か`lastValueFrom`へ移行します。

```ts
// 旧: toPromise（RxJS 7では非推奨）
const value = await source$.toPromise();
```

```ts
// 新: 最初の値か最後の値かを明示する
import { firstValueFrom, lastValueFrom } from 'rxjs';

const first = await firstValueFrom(source$);
const last = await lastValueFrom(source$);
```

最初の値がほしいなら`firstValueFrom`、最後の値がほしいなら`lastValueFrom`を使います。詳しくは「特殊なObservableとPromise相互変換」の章で扱いました。

## retryWhenからretryの設定オブジェクトへ

再試行の間隔を制御する`retryWhen`は、RxJS 7.8で非推奨です。新しいコードでは、`retry`に設定オブジェクトを渡します。

```ts
// 旧: retryWhen（非推奨）
source$.pipe(
  retryWhen((errors) => errors.pipe(delay(1000))),
);
```

```ts
// 新: retryのdelayオプション
import { retry, timer } from 'rxjs';

source$.pipe(
  retry({ count: 3, delay: () => timer(1000) }),
);
```

## multicast・publish・refCountからshare・connectableへ

共有まわりのOperatorは、`multicast`、`publish`、`refCount`など種類が多く、組み合わせも複雑でした。現在は`share`と`connectable`に整理されています。

```ts
// 旧: publish + refCount
source$.pipe(publish(), refCount());
```

```ts
// 新: share
import { share } from 'rxjs';

source$.pipe(share());
```

多くの場合、`share`か`shareReplay`で置き換えられます。詳しくは「shareとshareReplay・Subjectによる状態管理」の章で扱いました。

## mapTo・switchMapToなどの〜To系から関数形式へ

`mapTo`や`switchMapTo`など、末尾に`To`が付くOperatorは、RxJS 7.8で非推奨です。値を直接渡す形から、関数を渡す形へ移ります。

```ts
// 旧: mapTo（非推奨）
source$.pipe(mapTo('固定値'));
```

```ts
// 新: mapに関数を渡す
import { map } from 'rxjs';

source$.pipe(map(() => '固定値'));
```

`switchMapTo`も同様に、`switchMap(() => inner$)`と書きます。関数を渡す形に統一されたことで、Operatorの種類が減り、覚えることが少なくなりました。

## 古いsubscribeの引数形式からObserverオブジェクトへ

`subscribe`に、`next`・`error`・`complete`の関数を順番に並べて渡す形は非推奨になりました。Observerオブジェクトを渡します。

```ts
// 旧: 関数を順番に渡す（非推奨）
source$.subscribe(
  (value) => console.log(value),
  (error) => console.error(error),
  () => console.log('done'),
);
```

```ts
// 新: Observerオブジェクトを渡す
source$.subscribe({
  next: (value) => console.log(value),
  error: (error) => console.error(error),
  complete: () => console.log('done'),
});
```

どの引数がどの通知なのかが名前でわかり、必要なものだけを書けます。

## rxjs/operatorsからルートimportへ

RxJS 7.2以降は、Operatorも`rxjs`本体からimportする形が推奨されています。`rxjs/operators`からのimportも7.8では動作するため、既存コードを急いで直す必要はありません。本書ではimport元をそろえるため、ルートimportを使います。

```ts
// RxJS 6〜7.1で一般的だった形（7.8でも動作する）
import { map, filter } from 'rxjs/operators';
```

```ts
// 新: rxjs から
import { map, filter } from 'rxjs';
```

## Nested SubscribeからFlattening Operatorへ

購読の中で購読するNested Subscribeは、Flattening Operatorへ置き換えます。

```ts
// 旧: Nested Subscribe
keyword$.subscribe((keyword) => {
  searchApi(keyword).subscribe((result) => {
    render(result);
  });
});
```

```ts
// 新: switchMapで平坦化
import { switchMap } from 'rxjs';

keyword$
  .pipe(switchMap((keyword) => searchApi(keyword)))
  .subscribe((result) => render(result));
```

置き換えの考え方は、「Higher-order ObservableとNested Subscribe」と「Flattening Operator」の章で扱いました。

## その他の見直しどころ

コードを読み直すときは、次の点も確認します。

- **手動のSubscription管理**: 個別に`unsubscribe`を並べているなら、`takeUntil`や`add`によるまとめを検討します。
- **shareReplayによる購読の残留**: 終わらないsourceに`shareReplay(1)`だけを指定すると、購読者が0でも共有元の購読が残ることがあります。`{ bufferSize: 1, refCount: true }`を検討します。
- **BehaviorSubjectの外部公開**: Subjectをそのまま公開しているなら、`asObservable`で読み取り専用にして公開します。

これらは本編でも触れました。古いコードを新しい書き方へ直すときの、チェックリストとして使ってください。
