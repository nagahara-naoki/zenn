---
title: "shareとshareReplay・Subjectによる状態管理"
---

前章では、SubjectでCold Observableを共有する、手作業のパターンを見ました。この章では、それをずっと簡単に書けるOperator、`share`と`shareReplay`を扱います。

さらに、`BehaviorSubject`を使った、小規模な状態管理も見ます。現在の状態を保持し、その変化を配信する、という状態管理の基本形です。あわせて、Subjectを安全に使うための作法と、状態管理ライブラリに切り替えるべき境界も確認します。

## shareで実行を共有する

「Cold ObservableをSubjectに流し込む」という、前章で書いたパターンは、手作業では少し面倒でした。これを、`share`Operatorは1行にまとめてくれます。

```ts
import { interval, share } from 'rxjs';

const shared$ = interval(1000).pipe(share());

shared$.subscribe((v) => console.log('A:', v));
shared$.subscribe((v) => console.log('B:', v));

// AとBは同じintervalを共有する（タイマーは1つだけ）
```

`share()`を付けるだけで、複数の購読者が1つの実行を共有します。手作業だったSubjectの受け渡しを、`share`が内部で、まるごと肩代わりしてくれるのです。だいぶ楽になりました。

## 購読者数とrefCount

`share`は、購読者の数を数えています。この仕組みを、refCount（参照カウント）と呼びます。

動きは、こうです。最初の購読者が現れたとき、共有元のObservableの実行を始めます。以降の購読者は、その実行に相乗りします。購読者が増えても、実行は1つのままです。

```mermaid
flowchart LR
  A["最初の購読者"] --> S["共有元の実行が始まる"]
  B["2人目の購読者"] --> S
  C["3人目の購読者"] --> S
```

## 最後の購読者が解除された場合

refCountは、購読者が減ったときも、数えています。すべての購読者が解除されて、購読者数が0になると、共有元の実行を止めます。

そして、そのあとに新しい購読者が現れると、実行はもう一度、最初から始まります。つまり`share`は、「誰かが見ているあいだだけ実行し、誰も見なくなったら止める」という、賢い動きをします。この性質のおかげで、誰も使っていない処理が動き続ける、という無駄を避けられます。

## shareReplayで最新値を再配信する

`share`には、1つ弱点があります。前章のSubjectと同じで、あとから購読した人は、それまでに流れた値を受け取れないのです。共有はしても、過去は配り直しません。

過去の値も、新しい購読者へ配りたいときは、`shareReplay`を使います。指定した数だけ過去の値を覚えておき、新しい購読者に配り直してくれます。

```ts
import { shareReplay, from } from 'rxjs';

const tasks$ = from(fetch('/api/tasks').then((r) => r.json())).pipe(
  shareReplay(1), // 直近1件を覚えて配り直す
);

tasks$.subscribe(renderList);
// あとから購読しても、再取得せずにキャッシュ済みの結果を受け取れる
setTimeout(() => tasks$.subscribe(updateCount), 3000);
```

## レスポンスのキャッシュ

この`shareReplay(1)`は、HTTPレスポンスのキャッシュに、とてもよく使われます。1回だけリクエストを送り、その結果を覚えておいて、あとから購読した人にも、同じ結果を配ります。

前章で見た「多重リクエスト」の問題を、`shareReplay(1)`はきれいに解決します。何人が購読しても、リクエストは1回で済み、全員が同じ結果を受け取れるからです。マスターデータのように、一度取れば当分変わらないデータの取得に、ぴったりです。

## 意図しない永続購読

便利な`shareReplay`ですが、大きな落とし穴があります。使い方によっては、すべての購読者が解除されても、共有元の購読が残り続けてしまうのです。

これは、`shareReplay`が既定では、購読者数に連動しないためです。共有元をずっと購読したままにするので、終わらないObservableを`shareReplay`すると、購読が永遠に残り、メモリリークになります。これは、初学者が気づきにくい、やっかいな落とし穴です。

防ぐには、購読者数に連動する設定を、明示的に指定します。

```ts
import { shareReplay } from 'rxjs';

source$.pipe(
  shareReplay({ bufferSize: 1, refCount: true }), // 購読者が0になったら止める
);
```

`refCount: true`を指定すると、`share`と同じく、購読者が0になったときに、共有元の購読を解除してくれます。終わらないObservableに`shareReplay`を使うときは、この設定を忘れないようにしてください。

## キャッシュの無効化

`shareReplay`によるキャッシュには、もう1つ知っておきたい点があります。キャッシュは、自動では新しくなりません。一度覚えた値を、ずっと配り続けます。

だから、データを取り直したいときは、キャッシュを持つObservableそのものを作り直すか、再取得のきっかけを別に用意します。たとえば、更新ボタンのクリックを起点にして、新しいObservableへ切り替える、といった方法です。`shareReplay`は「1回取って共有する」のは得意ですが、「定期的に最新に保つ」のは、自動ではしない。この点を覚えておいてください。

## BehaviorSubjectで状態を保持する

ここからは、状態管理に話を移します。`BehaviorSubject`は、前章で見たとおり、現在の値を保持できるので、小規模な状態管理に向いています。

```ts
import { BehaviorSubject } from 'rxjs';

const count$ = new BehaviorSubject(0);

count$.subscribe((value) => console.log('現在の値:', value));

count$.next(count$.value + 1); // 1
count$.next(count$.value + 1); // 2
```

`BehaviorSubject`は、現在の値を`value`で読め、`next`で更新できます。そして、購読すると、まず現在の値がすぐに届きます。「状態を保持し、変化を配信する」という、状態管理の基本形が、これだけで作れます。

## Subjectを外部へ公開しない

状態管理でSubjectを使うときには、守るべき作法があります。Subjectそのものを、外部へ公開しないことです。

なぜでしょうか。Subjectを公開してしまうと、どこからでも`next`で値を流せてしまい、状態の変化を追えなくなるからです。「外から自由に流せる」という、前章で触れた性質が、ここでは裏目に出ます。そこで、外へは「読み取り専用のObservable」だけを公開し、更新は「決められたメソッド」を通してのみ行えるようにします。

```ts
import { BehaviorSubject } from 'rxjs';

class CounterStore {
  private readonly state$ = new BehaviorSubject(0);

  // 外へは読み取り専用のObservableだけを公開する
  readonly count$ = this.state$.asObservable();

  increment() {
    this.state$.next(this.state$.value + 1);
  }

  reset() {
    this.state$.next(0);
  }
}
```

`asObservable`でSubjectをObservableに変えて公開すると、外からは購読できても、`next`は呼べません。状態を変えられるのは、`increment`や`reset`といったメソッドだけになります。こうして、状態を変える経路が1本にまとまり、変化を追いやすくなります。

## scanによる状態更新

もう1つの書き方が、`scan`を使う方法です。操作（アクション）のストリームを`scan`に通し、前の状態と操作から、新しい状態を作ります。

```ts
import { Subject, scan, startWith } from 'rxjs';

type Action = { type: 'increment' } | { type: 'reset' };

const action$ = new Subject<Action>();

const count$ = action$.pipe(
  scan((state, action) => {
    switch (action.type) {
      case 'increment':
        return state + 1;
      case 'reset':
        return 0;
    }
  }, 0),
  startWith(0),
);
```

「操作を流すと、状態が更新されて流れてくる」という形です。「値を変換・選別・蓄積する」の章で見た`scan`のイミュータブルな更新が、ここで生きてきます。そして、この「操作→状態更新」という考え方は、実はNgRxなどの状態管理ライブラリの、土台になっているものです。本書で学んだ部品が、より大きな仕組みにつながっているわけです。

## Subjectを乱用しない、状態管理ライブラリの境界

`BehaviorSubject`による状態管理は、小さな範囲では十分に使えます。1つの画面の中の状態や、いくつかのコンポーネントで共有する程度の状態なら、これで足ります。

しかし、状態が増え、更新の経路が複雑になってくると、手作りのSubject管理では、だんだん追いにくくなります。状態の変化を記録したい、複数の状態を組み合わせたい、といった要求が出てきたら、専用の状態管理ライブラリを検討する時期です。Angularなら、シリーズ後続で扱うNgRxが、その選択肢になります。「RxJSでどこまで状態を扱い、どこからライブラリに任せるか」。その境界を意識すると、コードを健全に保てます。

Subjectは、状態管理でも便利な道具ですが、乱用は禁物です。前章でも触れたとおり、まずObservableとOperatorで書けないかを考え、状態の保持や共有が本当に必要なときにだけ、Subjectを使ってください。

## まとめ

この章では、共有のOperatorと状態管理を確認しました。

- `share`は、複数の購読者で1つの実行を共有します。refCountで購読者数を数えます。
- `share`は購読者が0になると実行を止め、次の購読で最初からやり直します。
- `shareReplay`は過去の値を配り直し、HTTPレスポンスのキャッシュに向きます。
- `shareReplay`は既定で購読が残り続けます。`refCount: true`を指定して防ぎます。
- `BehaviorSubject`で状態を保持し、`asObservable`で読み取り専用にして公開します。
- 小規模ならSubjectで足りますが、複雑になったら状態管理ライブラリを検討します。

次章からは、ストリームを壊れにくくするための仕組みを扱います。まず、エラー処理・再試行・キャンセルです。
