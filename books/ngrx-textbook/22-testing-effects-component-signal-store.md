---
title: "Effects・Component・SignalStoreをテストする"
---

前章では、純粋関数であるReducerとSelectorをテストしました。純粋関数は身軽なので、テストも簡単でした。この章では、もう少し手のかかる部分、つまり副作用や状態を持つ部分のテストを扱います。

Effectsは、Actionを受けてActionを流す非同期処理です。ComponentはStoreから状態を読み、利用者の操作をActionへ変換します。SignalStoreは、状態とメソッドを持ちます。いずれも純粋関数ほど単純ではありませんが、NgRxにはテスト用の道具が用意されています。それらを使って、副作用、画面とStoreの接続、状態の振る舞いを検証していきます。

Effects、Component、SignalStoreの順に見ていきます。

## Effectsのテストで確かめること

Effectのテストで確かめたいのは、シンプルです。「あるActionを受けたとき、期待したActionを流すか」です。たとえば「`loadTasks`を受けてAPIが成功したら、`loadTasksSuccess`を流す」ことを検証します。

そのために、2つのものを用意します。1つは、Effectに流し込むActionのストリーム。もう1つは、本物のサーバーを呼ばないための、API通信のモック（偽物）です。Actionのストリームは、NgRxが用意する`provideMockActions`で作り、APIはテスト用の偽の実装に差し替えます。テストのたびに本物のサーバーを呼んでいたら、遅いうえに不安定なので、モックに置き換えるわけです。

## provideMockActionsでEffectをテストする

`@ngrx/effects/testing`の`provideMockActions`を使って、Effectに流すActionのストリームを差し込みます。

```ts:src/app/tasks/tasks.effects.spec.ts
import { TestBed, runInInjectionContext } from '@angular/core/testing';
import { provideMockActions } from '@ngrx/effects/testing';
import { of, Observable } from 'rxjs';
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { loadTasks } from './tasks.effects';
import { tasksActions } from './tasks.actions';
import { TaskApi } from './task-api';

describe('loadTasks effect', () => {
  let actions$: Observable<unknown>;
  let api: { getTasks: ReturnType<typeof vi.fn> };

  beforeEach(() => {
    api = { getTasks: vi.fn() };
    TestBed.configureTestingModule({
      providers: [
        provideMockActions(() => actions$),
        { provide: TaskApi, useValue: api },
      ],
    });
  });

  it('成功したらloadTasksSuccessを流す', (done) => {
    const tasks = [{ id: '1', title: 'A', completed: false }];
    api.getTasks.mockReturnValue(of(tasks)); // APIが成功を返すように仕込む
    actions$ = of(tasksActions.loadTasks()); // loadTasksを流す

    runInInjectionContext(TestBed, () => {
      loadTasks().subscribe((action) => {
        expect(action).toEqual(tasksActions.loadTasksSuccess({ tasks }));
        done();
      });
    });
  });
});
```

流れを追います。準備として、`actions$`に`loadTasks`を流し、APIのモックが成功（`of(tasks)`）を返すように仕込みます。そのうえでEffectを購読し、流れてきたActionが`loadTasksSuccess`であることを確かめます。Functional Effectは関数なので、依存を注入した状態で呼び出し、購読するだけでテストできます。

失敗のテストも、同じ形で書けます。APIのモックがエラーを返すようにして、`loadTasksFailure`が流れることを確かめます。第13章「Effectのエラー・キャンセル・状態参照」で学んだ「`catchError`をInnerに置く」設計が正しくできていれば、この失敗テストがきちんと通ります。テストが、設計の正しさを裏づけてくれるわけです。

## marbleテストで時間を検証する

時間がからむEffectや、通信の順序を厳密に検証したいときは、marbleテストという手法を使います。`provideMockActions`は、`hot`や`cold`で書いたActionストリームと組み合わせられます。

marbleテストの記法そのものは、『RxJSの教科書』のテストの章で扱いました。NgRxのEffectでも、同じ`TestScheduler`とmarble記法を使って、Actionが流れる時刻まで細かく検証できます。ただ、最初からmarbleに挑む必要はありません。まずは`of`を使った素朴なテストから始め、時間や順序の検証が本当に必要になったら、marbleへ進む、という順で十分です。

## ComponentとStoreの接続をテストする

NgRxを使うComponentでは、業務ロジックそのものより、Storeとの接続を確かめます。主な対象は「Selectorから受け取った状態を表示できるか」と「利用者の操作に応じて正しいActionをdispatchするか」の2点です。

Classic Storeに依存するComponentは、`@ngrx/store/testing`の`provideMockStore()`でStoreを差し替えられます。本物のReducerやEffectsを起動せず、Selectorの返り値だけを指定できるため、Componentの責務に絞って検証できます。

ここでは、タスク一覧を表示し、ボタンが押されたら読み込みActionを流すComponentを対象にします。

```ts:src/app/tasks/task-list.component.ts
import { Component, inject } from '@angular/core';
import { Store } from '@ngrx/store';
import { tasksActions } from './tasks.actions';
import { tasksFeature } from './tasks.reducer';

@Component({
  selector: 'app-task-list',
  template: `
    <button type="button" (click)="reload()">再読み込み</button>
    @for (task of tasks(); track task.id) {
      <p>{{ task.title }}</p>
    }
  `,
})
export class TaskListComponent {
  private readonly store = inject(Store);
  readonly tasks = this.store.selectSignal(tasksFeature.selectTasks);

  reload(): void {
    this.store.dispatch(tasksActions.loadTasks());
  }
}
```

```ts:src/app/tasks/task-list.component.spec.ts
import { TestBed } from '@angular/core/testing';
import { MockStore, provideMockStore } from '@ngrx/store/testing';
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { TaskListComponent } from './task-list.component';
import { tasksActions } from './tasks.actions';
import { tasksFeature } from './tasks.reducer';

describe('TaskListComponent', () => {
  const tasks = [{ id: '1', title: 'A', completed: false }];

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [TaskListComponent],
      providers: [
        provideMockStore({
          selectors: [
            { selector: tasksFeature.selectTasks, value: tasks },
          ],
        }),
      ],
    });
  });

  it('タスクを表示し、再読み込みでActionを流す', () => {
    const fixture = TestBed.createComponent(TaskListComponent);
    const store = TestBed.inject(MockStore);
    const dispatch = vi.spyOn(store, 'dispatch');

    fixture.detectChanges();

    expect(fixture.nativeElement.textContent).toContain('A');

    fixture.nativeElement.querySelector('button').click();
    expect(dispatch).toHaveBeenCalledWith(tasksActions.loadTasks());
  });
});
```

`provideMockStore()`の`selectors`へ、Componentが読むSelectorとテスト用の値を渡します。`fixture.detectChanges()`のあとに表示を確認し、ボタン操作では`dispatch`されたActionを検証します。ここでReducerの状態更新やEffectのAPI通信まで試す必要はありません。それらは、それぞれの単体テストが担当します。

ComponentがSignalStoreを直接注入する場合は、公開APIと同じ形を持つテスト用オブジェクトへ差し替える方法も使えます。状態や派生値には`signal()`、メソッドには`vi.fn()`を用意し、Component側から見える契約だけを再現します。

## SignalStoreのテストで確かめること

次はSignalStoreです。ここで確かめたいのは、「メソッドを呼んだとき、状態のSignalが期待どおりに変わるか」です。

SignalStoreは、DIの仕組みに乗っているので、`TestBed`で注入して使います。Storeを取り出し、メソッドを呼び、Signalを読んで、値を確かめる。この流れです。依存を差し替えたいときも、TestBedで行えます。

## SignalStoreをテストする

`TestBed.inject`でStoreを取り出し、メソッドを呼んで、Signalの値を検証します。

```ts:src/app/tasks/tasks.store.spec.ts
import { TestBed } from '@angular/core/testing';
import { describe, it, expect } from 'vitest';
import { TasksStore } from './tasks.store';

describe('TasksStore', () => {
  it('setFilterで絞り込み条件が変わる', () => {
    const store = TestBed.inject(TasksStore);

    store.setFilter('active');

    expect(store.filter()).toBe('active');
  });

  it('addTaskで一覧に追加される', () => {
    const store = TestBed.inject(TasksStore);

    store.addTask('新しいタスク');

    expect(store.tasks().length).toBe(1);
    expect(store.tasks()[0].title).toBe('新しいタスク');
  });
});
```

メソッド（`setFilter`や`addTask`）を呼んだあと、`store.filter()`や`store.tasks()`というSignalを読んで、値を確かめます。派生値（`withComputed`で定義したもの）も、同じくSignalとして読めるので、計算結果を検証できます。Classic StoreのReducerテストと比べると、Storeのインスタンスを注入する点が違いますが、「操作して、結果を確かめる」という骨格は同じです。

## unprotectedで状態を差し込む

テストでは、「特定の状態から始めたい」ことがよくあります。たとえば「タスクが2件ある状態で、絞り込みが正しく動くか」を試したい場合です。しかし、SignalStoreの状態は、外部から勝手に書き換えられないよう、守られています。テストのときだけ、この守りを外すには、`@ngrx/signals/testing`の`unprotected`を使います。

```ts
import { patchState } from '@ngrx/signals';
import { unprotected } from '@ngrx/signals/testing';

it('完了タスクだけを絞り込める', () => {
  const store = TestBed.inject(TasksStore);

  // テスト用に、初期状態を直接差し込む
  patchState(unprotected(store), {
    tasks: [
      { id: '1', title: 'A', completed: false },
      { id: '2', title: 'B', completed: true },
    ],
    filter: 'completed',
  });

  expect(store.visibleTasks()).toEqual([{ id: '2', title: 'B', completed: true }]);
});
```

`unprotected(store)`を`patchState`に渡すと、テストの中でだけ、状態を直接組み立てられます。ここでは、タスク2件と`filter: 'completed'`をあらかじめ差し込み、`visibleTasks`が完了タスクだけを返すことを確かめています。もちろん、この`unprotected`はテスト専用で、本番のコードでは、状態は守られたままです。

## 非同期メソッドのテスト

`rxMethod`で書いた非同期メソッドは、APIのモックを差し替えてテストします。メソッドを呼び、状態が正しく更新されるのを確かめる、という流れです。

具体的には、依存のAPIをTestBedで偽の実装に差し替え、成功を返すように仕込んでから、`store.loadTasks()`を呼びます。呼び出したあとに、`store.tasks()`や`store.loading()`が期待どおりになっているかを検証します。同期的にすぐ完了するモックを使えば、待ち時間なしにテストできます。ComponentStoreの`effect`のテストも、同じ考え方で書けます。

## まとめ

この章では、Effects・Component・SignalStoreのテストを確認しました。

- Effectのテストは、`provideMockActions`でActionを流し、APIをモックして、流れるActionを検証します。
- Functional Effectは関数なので、依存を注入して購読するだけでテストできます。
- 時間や順序を厳密に検証するなら、marbleテストを使います。
- Componentは`provideMockStore()`でStoreを差し替え、Selectorからの表示とActionのdispatchを検証します。
- SignalStoreは`TestBed.inject`で注入し、メソッドを呼んでSignalの値を確かめます。
- `@ngrx/signals/testing`の`unprotected`で、テスト用に状態を直接差し込めます。
- 非同期メソッドは、APIをモックし、呼び出し後の状態を検証します。

次章からは、本書の締めくくりです。まず、NgRxでやりがちなアンチパターンを整理します。
