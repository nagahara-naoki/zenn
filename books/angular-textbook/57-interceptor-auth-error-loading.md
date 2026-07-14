---
title: "第47章 Interceptor・認証・エラー・ローディング設計"
---

第9部の締めくくりとして、通信にまつわる横断的な設計を学びます。個々の通信そのものは、前章までで扱いました。しかし実務では、「すべての通信に認証トークンを添える」「通信の失敗を一か所でまとめて処理する」「通信中はローディングを表示する」といった、通信全体に共通する処理が必要になります。

これらを、通信のたびに個別に書くのは、繰り返しが多く、書き忘れも生じます。そこで役立つのが、Interceptor（インターセプター）です。Interceptorは、すべてのHTTP通信の途中に割り込み、共通の処理を差し込む仕組みです。この章では、Interceptorを軸に、認証・エラー・ローディングという、通信設計の定番テーマを扱います。

:::message
**この章で学ぶこと**

- Interceptorによる通信への割り込み
- 認証トークンの付与
- エラーの共通処理
- ローディング状態の管理
:::

## Interceptorとは

Interceptorは、アプリケーションが送るすべてのHTTP通信の途中に割り込む仕組みです。要求がサーバーへ送られる前、あるいは応答が返ってきた後に、共通の処理を差し込めます。「通信の関所」のようなものだと考えてください。すべての通信が、この関所を通ります。

モダンAngularのInterceptorは、関数として書きます。第24章で学んだ`inject()`があるおかげで、関数の中で依存を受け取れます。`HttpInterceptorFn`という型の関数を定義します。

```ts:src/app/logging-interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  console.log('送信:', req.url);
  return next(req); // 次の処理へ要求を渡す
};
```

`req`が送られる要求、`next`が次の処理へ渡す関数です。`next(req)`を呼ぶことで、要求が次へ進みます。ここでは、送信前にログを出しています。作ったInterceptorは、`provideHttpClient()`に`withInterceptors()`で登録します。

```ts:src/app/app.config.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([loggingInterceptor])),
  ],
};
```

これで、すべての通信が`loggingInterceptor`を通るようになります。複数のInterceptorを配列で登録でき、順に適用されます。かつてはクラスとして書いていましたが、関数型のほうが簡潔で、`inject()`とも自然になじむため、現在は関数型が標準です。

## 認証トークンを付与する

Interceptorのもっとも典型的な用途が、認証です。多くのAPIは、ログイン済みであることを示すトークンを、要求のヘッダーに添えることを求めます。これをすべての通信に手作業で付けるのは大変ですが、Interceptorなら一か所で済みます。

```ts:src/app/auth-interceptor.ts
import { inject } from '@angular/core';
import { HttpInterceptorFn } from '@angular/common/http';
import { AuthService } from './auth';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  const token = auth.getToken();

  if (token) {
    // トークンをヘッダーに添えた要求を作り直す
    const authReq = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` },
    });
    return next(authReq);
  }
  return next(req);
};
```

`inject(AuthService)`で認証Serviceを受け取り、トークンを取り出します。要求は変更できないため、`req.clone()`で、ヘッダーを添えた新しい要求を作り、それを`next`へ渡します。これで、すべての通信に自動でトークンが付きます。認証の仕組みを一か所に集約でき、各通信のコードは認証を意識せずに済みます。

## エラーを共通で処理する

通信は失敗することがあります。ネットワークの問題、サーバーのエラー、認証切れなど、原因はさまざまです。これらのエラー処理を、通信ごとに書くのではなく、Interceptorでまとめて扱えます。第39章で学んだ`catchError`を、ここで使います。

```ts:src/app/error-interceptor.ts
import { inject } from '@angular/core';
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { Router } from '@angular/router';
import { catchError, throwError } from 'rxjs';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        router.navigate(['/login']); // 認証切れならログインへ
      }
      return throwError(() => error);
    }),
  );
};
```

`next(req)`が返すObservableに`catchError`をつなぎ、エラーを捉えます。ここでは、認証切れを示す`401`なら、ログインページへ遷移させています。共通のエラー処理を一か所に集約でき、個々の通信は、それぞれ固有のエラー処理だけに集中できます。ログの記録や、利用者への通知も、この場所で共通化できます。

## ローディング状態を管理する

「通信中はローディング表示を出す」という要求も、よくあります。これも、Interceptorで、進行中の通信の数を数えることで実現できます。通信が始まったら数を増やし、終わったら減らす。数が0より大きいあいだ、ローディング中とみなします。

```ts:src/app/loading-interceptor.ts
import { inject } from '@angular/core';
import { HttpInterceptorFn } from '@angular/common/http';
import { finalize } from 'rxjs';
import { LoadingService } from './loading';

export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
  const loading = inject(LoadingService);
  loading.start(); // 通信開始：カウントを増やす

  return next(req).pipe(
    finalize(() => loading.stop()), // 成否によらず終了：カウントを減らす
  );
};
```

第38章で学んだ`finalize`が、ここで活きます。通信が成功しても失敗しても、`finalize`で確実にカウントを減らせるため、ローディングが消えずに残る、という不具合を防げます。`LoadingService`は、進行中の数をSignalで持ち、画面はそのSignalを見てローディング表示を出す、という設計にすれば、モダンAngularらしくまとまります。

## 通信設計の全体像

ここまでの要素を組み合わせると、通信の横断的な関心事が、Interceptorという一か所に整理されます。

- 認証トークンの付与（`authInterceptor`）
- エラーの共通処理（`errorInterceptor`）
- ローディングの管理（`loadingInterceptor`）

これらをそれぞれ独立したInterceptorとして書き、`withInterceptors([...])`に並べて登録します。各Interceptorが単一の関心事に集中するため、見通しがよく、テストもしやすくなります。第22章で学んだ責務の分離が、通信の設計にも貫かれているのがわかります。個々の通信コードは本来の目的だけに集中でき、共通処理はInterceptorが引き受ける。この分業が、保守しやすい通信層を作ります。

## よくあるつまずき

- **`next(req)`の呼び忘れ**: Interceptorの中で`next(req)`を呼ばないと、要求が次へ進まず、通信が止まります。処理の最後に、必ず`next`の結果を返します。
- **要求を直接書き換えようとする**: 要求（`req`）は変更できません。ヘッダーなどを足すときは、`req.clone()`で新しい要求を作ります。
- **Interceptorに多くを詰め込む**: 1つのInterceptorに認証もエラーもローディングも書くと、見通しが悪くなります。関心事ごとに分け、`withInterceptors([...])`に並べます。
- **登録順を意識しない**: Interceptorは登録した順に適用されます。処理の順序が重要な場合（例えばトークン付与とログ出力の順）は、配列の並び順に注意します。

## 通信設計とSignalの調和

最後に、この章の設計が、本書で学んできたモダンAngularの考え方と、いかに調和しているかを確認します。Interceptorは関数型で`inject()`を使い、ローディング状態は`LoadingService`がSignalで持ち、通信結果は`httpResource()`や`toSignal()`でSignalとして扱う。関数中心・Signal中心という一貫した方針が、通信という実務的な領域にも貫かれています。個々の技術がばらばらにあるのではなく、ひとつの設計思想のもとで噛み合っていることを感じ取ってください。この一貫性こそ、モダンAngularを学ぶ価値のひとつです。

## まとめ

- Interceptorは、すべてのHTTP通信に割り込んで共通処理を差し込む仕組みです
- 関数型（`HttpInterceptorFn`）で書き、`withInterceptors()`で登録します
- 認証トークンの付与は、`req.clone()`でヘッダーを添えて実現します
- エラーの共通処理は`catchError`で、ローディング管理は`finalize`で行います
- 横断的な関心事をInterceptorに集約すると、個々の通信コードが本来の目的に集中できます

以上で第9部は終わりです。次の第10部では、アプリケーション全体の状態をどう管理するか、その設計とNgRxを学びます。
