---
title: "API通信とFlattening Operatorの選択"
---

前章で、Effectの基本の形を見ました。この章では、Effectのいちばんの仕事であるAPI通信を掘り下げます。

Effectの中では、Actionを受けてAPIを呼びます。このとき避けて通れないのが、「どのFlattening Operatorで平坦化するか」という選択です。`switchMap`、`concatMap`、`mergeMap`、`exhaustMap`。名前は似ていますが、選び方を誤ると、通信が二重に走ったり、途中でキャンセルされたりと、実際の不具合につながります。

Flattening Operatorそのものの仕組みは『RxJSの教科書』で詳しく扱いました。この章では、それをNgRxのEffectという具体的な場面に当てはめ、どの通信にどれを選ぶかを考えます。

## Effect内のAPI通信の形

API通信を行うEffectは、いつも決まった形をしています。要求のActionを受け、Flattening OperatorでAPIを呼び、その結果を成功・失敗のActionに変える、という形です。

```ts
actions$.pipe(
  ofType(tasksActions.loadTasks),
  フラット化Operator(() =>
    api.getTasks().pipe(
      map((tasks) => tasksActions.loadTasksSuccess({ tasks })),
      catchError((error) =>
        of(tasksActions.loadTasksFailure({ error: error.message })),
      ),
    ),
  ),
);
```

この「フラット化Operator」と書いた部分に、`switchMap`などが入ります。ここに何を入れるかが、この章のテーマです。

## なぜ平坦化が必要か

そもそも、なぜ平坦化が必要なのでしょうか。少し立ち止まって考えます。

`ofType`を通ったあとのストリームには、Actionが流れています。ここで`map`を使ってAPIを呼ぶと、`api.getTasks()`はObservable（通信の結果を流すストリーム）を返すので、「Observableを流すObservable」という入れ子ができてしまいます。この入れ子のままでは、通信の結果を受け取れません。入れ子をほどいて、通信の結果を1本のストリームに流し込む必要があります。これが平坦化です。

そして、要求のActionが立て続けに届いたとき、進行中の通信をどう扱うかで、選ぶOperatorが変わります。前の通信を止めるのか、終わるまで待たせるのか、並行して走らせるのか、それとも無視するのか。この判断が、Flattening Operatorの選択そのものです。1つずつ見ていきます。

## 読み込みにはswitchMap

一覧の読み込みや検索のように、「最新の要求の結果だけがほしい」通信には、`switchMap`を使います。新しい要求が来たら、前の通信を解除して、新しい通信に乗り換えます。

```ts
export const loadTasks = createEffect(
  (actions$ = inject(Actions), api = inject(TaskApi)) =>
    actions$.pipe(
      ofType(tasksActions.loadTasks),
      switchMap(() =>
        api.getTasks().pipe(
          map((tasks) => tasksActions.loadTasksSuccess({ tasks })),
          catchError((error) =>
            of(tasksActions.loadTasksFailure({ error: error.message })),
          ),
        ),
      ),
    ),
  { functional: true },
);
```

たとえば、絞り込み条件を次々に変えて何度も読み込むとします。`switchMap`なら、新しい読み込みが始まった時点で古い読み込みは解除されるので、古い結果が新しい結果を上書きすることがありません。つねに、最後に要求した結果だけが状態に反映されます。

## 保存・更新にはconcatMapかexhaustMap

一方、保存や更新のような「書き込み」には、`switchMap`は向きません。理由を考えてみてください。書き込みの途中で次の書き込みが来たとき、`switchMap`は前の書き込みを解除してしまいます。すると、保存が完了しないまま中断され、データが中途半端な状態になるおそれがあります。読み込みなら途中でやめても問題ありませんが、書き込みを途中でやめるのは危険です。

書き込みを、順番に、もれなく実行したいなら`concatMap`を使います。前の書き込みが終わってから、次を実行します。

```ts
export const updateTask = createEffect(
  (actions$ = inject(Actions), api = inject(TaskApi)) =>
    actions$.pipe(
      ofType(tasksActions.updateTask),
      concatMap(({ task }) =>
        api.updateTask(task).pipe(
          map((updated) => tasksActions.updateTaskSuccess({ task: updated })),
          catchError((error) =>
            of(tasksActions.updateTaskFailure({ error: error.message })),
          ),
        ),
      ),
    ),
  { functional: true },
);
```

二重送信を防ぎたいなら`exhaustMap`を使います。処理中に届いた要求を無視するので、送信ボタンを連打しても、通信は1回しか走りません。

```ts
export const submitOrder = createEffect(
  (actions$ = inject(Actions), api = inject(OrderApi)) =>
    actions$.pipe(
      ofType(orderActions.submit),
      exhaustMap(() =>
        api.submit().pipe(
          map(() => orderActions.submitSuccess()),
          catchError((error) =>
            of(orderActions.submitFailure({ error: error.message })),
          ),
        ),
      ),
    ),
  { functional: true },
);
```

## 独立した処理にはmergeMap

順番も最新も気にせず、それぞれを独立して並行実行したいなら`mergeMap`を使います。たとえば、選んだ複数のタスクをまとめて削除する、といった処理です。

```ts
export const removeTask = createEffect(
  (actions$ = inject(Actions), api = inject(TaskApi)) =>
    actions$.pipe(
      ofType(tasksActions.removeTask),
      mergeMap(({ id }) =>
        api.deleteTask(id).pipe(
          map(() => tasksActions.removeTaskSuccess({ id })),
          catchError((error) =>
            of(tasksActions.removeTaskFailure({ id, error: error.message })),
          ),
        ),
      ),
    ),
  { functional: true },
);
```

削除は、対象ごとに独立しています。タスクAの削除とタスクBの削除は、どちらが先に終わっても構いません。だから、並行して走らせてよい`mergeMap`が向いています。

## 選択をまとめる

Effectでの通信の種類と、選ぶOperatorを表にまとめます。

| 通信の種類 | Operator | 理由 |
|---|---|---|
| 一覧の読み込み・検索 | `switchMap` | 最新の要求の結果だけが必要 |
| 保存・更新（順序が重要） | `concatMap` | もれなく順番に実行する |
| 送信（二重送信を防ぐ） | `exhaustMap` | 処理中の要求を無視する |
| 独立した処理（一括削除など） | `mergeMap` | 並行して実行してよい |

迷ったときの判断の順序は、こうです。まず「最新の結果だけでよいか」を考えます。読み込みや検索ならその通りなので、`switchMap`です。書き込みなら違うので、次に「順序が重要か」「二重送信を防ぎたいか」を考え、`concatMap`か`exhaustMap`を選びます。どれにも当てはまらず、独立して並行させてよいなら`mergeMap`です。

ひとまずの合言葉は、「読み込みは`switchMap`、書き込みは`concatMap`か`exhaustMap`」です。これを出発点にすれば、大きく外すことはありません。

## まとめ

この章では、Effect内のAPI通信とFlattening Operatorの選択を確認しました。

- Effectでは、要求のActionを受け、APIを呼び、成功・失敗のActionに変えます。
- API呼び出しはObservableの入れ子になるため、平坦化が必要です。
- 読み込み・検索は`switchMap`で、最新の結果だけを残します。
- 保存・更新は`concatMap`で順番に、二重送信の防止は`exhaustMap`で行います。
- 独立した並行処理には`mergeMap`を使います。
- 「最新だけか、書き込みか」を起点に、選ぶOperatorを判断します。

次章では、Effectのエラー処理・キャンセル・状態参照を扱います。`catchError`を置く位置や、Effectが止まってしまう問題を見ていきます。
