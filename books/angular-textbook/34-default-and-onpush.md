---
title: "第27章 Default Change DetectionとOnPush"
---

前章で、変更検知がComponentツリーを上から下へたどって画面を更新することを学びました。この「どのようにツリーを確認するか」には、実は2つの戦略があります。すべてのComponentを毎回確認する戦略と、必要なComponentだけを確認する戦略です。前者を長らくDefault（既定）と呼び、後者をOnPushと呼んできました。

この2つの戦略の違いは、アプリケーションのパフォーマンスに直結します。そして重要なことに、Angular 22（2026年）で、この既定が切り替わりました。新しく作るComponentは、OnPushが標準になったのです。この章では、2つの戦略の違いと、なぜOnPushが標準になったのかを掘り下げます。

:::message
**この章で学ぶこと**

- 2つの変更検知戦略の違い
- OnPushが変更検知を走らせる条件
- Angular 22での既定の変化（OnPushが標準に）
- OnPushとSignal・不変性の関係
:::

## 2つの変更検知戦略

まず、従来から標準だった戦略を見ます。この戦略では、変更検知のサイクルが走るたびに、原則としてすべてのComponentを確認します。どこかで何かが起きて変更検知が始まると、ツリー全体をたどり、一つひとつのComponentのバインディングを検査するのです。

この戦略は、確実です。どこで状態が変わっても、必ず全体を確認するため、更新のもれが起きません。その代わり、確認の対象が常にツリー全体なので、無駄も生じます。実際には何も変わっていないComponentまで、毎回検査することになるためです。小さなアプリケーションでは問題になりませんが、Componentが数百に及ぶ規模では、この無駄が積み重なって、動作の重さにつながります。

もうひとつの戦略が、OnPushです。OnPushを指定したComponentは、「特定の条件が満たされたときだけ」確認の対象になります。何も起きていないなら、たとえ変更検知のサイクルが走っても、そのComponentは検査をとばされます。確認の範囲を絞ることで、無駄を大きく減らせるのです。

## OnPushが変更検知を走らせる条件

では、OnPushのComponentは、どんなときに確認されるのでしょうか。おもな条件は次の4つです。

- **入力（`input`）の参照が変わったとき**: 親から渡される入力が、新しい値（新しい参照）に変わった場合
- **Component自身やその子でイベントが起きたとき**: クリックなど、そのComponent内のイベントが発火した場合
- **テンプレート内の`async`パイプが新しい値を受け取ったとき**: 第16章で学んだ`async`が値を流した場合
- **明示的に更新を要求したとき**: `ChangeDetectorRef`の`markForCheck()`を呼んだ場合

これらはいずれも、「そのComponentの状態が実際に変わった可能性が高い」タイミングです。OnPushは、この確度の高いタイミングだけに確認を絞ります。逆にいえば、これらの条件を満たさずに状態を変えても、OnPushのComponentは更新されません。ここが、OnPushを使ううえで理解しておくべき勘所です。

とくに1つ目の「入力の参照が変わったとき」は重要です。前章で、オブジェクトや配列は参照で比較されると述べました。OnPushのComponentに配列を渡している場合、その配列の中身を書き換えても、参照が同じなら「変わっていない」とみなされ、更新されません。中身ではなく、新しい配列に差し替えることで、はじめて参照が変わり、更新が起きます。

この挙動を、具体例で確かめます。OnPushの子Componentに、商品の配列を渡すとします。

```ts:src/app/product-list.ts
@Component({
  selector: 'app-product-list',
  template: `@for (p of products(); track p.id) { <p>{{ p.name }}</p> }`,
})
export class ProductList {
  products = input.required<Product[]>();
}
```

親側で、次のように配列の中身を直接書き換えても、OnPushの子は更新されません。

```ts
// 更新されない：同じ配列の参照のまま中身を足している
addProduct(): void {
  this.items().push(newProduct);
}
```

一方、新しい配列に差し替えれば、参照が変わり、子が更新されます。

```ts
// 更新される：新しい配列に差し替えて参照を変える
addProduct(): void {
  this.items.update((list) => [...list, newProduct]);
}
```

`items`をSignalで持ち、`update()`で新しい配列に差し替えているため、参照が変わり、変化が確実に伝わります。OnPushと不変な更新、そしてSignalが、いかに噛み合っているかが見て取れます。「中身を書き換える」のではなく「新しい値に差し替える」という習慣が、OnPush時代の基本作法です。

## Angular 22での既定の変化

ここまで「従来の標準」「OnPush」と呼び分けてきましたが、Angular 22で、この関係が変わりました。

Angular 22より前は、Componentに変更検知戦略を指定しないと、全体を確認する戦略（`ChangeDetectionStrategy.Default`）が適用されていました。OnPushを使うには、明示的に指定する必要がありました。

```ts:Angular 22より前の書き方（明示的にOnPushを指定）
import { ChangeDetectionStrategy, Component } from '@angular/core';

@Component({
  selector: 'app-user-card',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `...`,
})
export class UserCard {}
```

Angular 22では、この既定が逆転しました。戦略を指定しないComponentは、OnPushとして扱われます。つまり、上記のような`changeDetection`の明示は、新規Componentでは不要になったのです。より効率のよいOnPushが、標準の振る舞いになりました。

これにともない、従来の「全体を確認する戦略」の名前も整理されました。この常時確認の戦略は`ChangeDetectionStrategy.Eager`という名前になり、`Default`は非推奨（deprecated）として残されつつ、`Eager`の別名という位置づけになりました。もし、あえて従来どおりの常時確認をさせたい場合は、`Eager`を明示します。

```ts:src/app/legacy-widget.ts
import { ChangeDetectionStrategy, Component } from '@angular/core';

@Component({
  selector: 'app-legacy-widget',
  changeDetection: ChangeDetectionStrategy.Eager, // 従来の常時確認
  template: `...`,
})
export class LegacyWidget {}
```

:::message
既存のアプリケーションをAngular 22へ更新するときは、互換性のために、明示的な戦略のないComponentへ自動で`Eager`が付与されます。これにより、更新しても従来の挙動が保たれます。新しく書くComponentは、既定のOnPushをそのまま活かすのがよいでしょう。
:::

## OnPushとSignal・不変性

OnPushは、状態の扱い方に一定の規律を求めます。「参照が変わったら更新する」という性質のため、状態を変えるときは、既存のオブジェクトを書き換えるのではなく、新しいオブジェクトに差し替える、という書き方が基本になります。この「変更のたびに新しい値を作る」考え方を、不変性（イミュータビリティ）と呼びます。

```ts
// OnPushと相性の悪い書き方（同じ配列の中身を書き換える）
this.items.push(newItem);

// OnPushと相性のよい書き方（新しい配列に差し替える）
this.items = [...this.items, newItem];
```

そして、この不変性を自然に扱えるのがSignalです。Signalの値を`set()`や`update()`で変えると、Angularは「そのSignalが変わった」ことを正確に把握します。参照の比較に頼らずとも、変化が確実に伝わるのです。Signalを使えば、OnPushのComponentでも、状態の変化が漏れなく画面に反映されます。

実のところ、OnPushが標準になった背景には、Signalの普及があります。Signalで状態を管理していれば、変更検知は「変わったSignalを使っているComponent」だけを、的確に更新できます。OnPushの「必要なところだけ確認する」という発想と、Signalの「どこで何が使われているかを追跡する」という性質は、非常に相性がよいのです。この組み合わせが、次章以降で学ぶZonelessへの道を開きます。

かつては、OnPushは「上級者が性能のために選ぶ、少し扱いの難しい戦略」と見られていました。参照の比較を意識せねばならず、慣れないうちは表示が更新されない不具合を招きやすかったためです。しかしSignalの登場で状況が変わりました。状態をSignalで持てば、参照の比較を意識せずとも変化が正しく伝わります。OnPushの難しさの多くが、Signalによって解消されたのです。OnPushが標準になったのは、こうしてOnPushが「特別な最適化」から「無理なく使える既定」へと変わったことの表れでもあります。

## よくあるつまずき

- **OnPushで表示が更新されない**: 多くは、オブジェクトや配列の中身を書き換えて、参照を変えていないことが原因です。新しい値に差し替えるか、Signalで状態を持つと解決します。
- **`Default`をそのまま使い続ける**: `Default`は非推奨になりました。従来の挙動が必要なら`Eager`を明示し、可能ならOnPush（既定）へ寄せていきます。
- **`markForCheck()`に頼りすぎる**: 手動で更新を要求する`markForCheck()`は最終手段です。まずはSignalや不変な更新で、自然に検知される形を目指します。手動呼び出しが増えてきたら、状態の持ち方を見直す合図だと捉えてください。

## まとめ

- 変更検知には、全体を確認する戦略と、必要な箇所だけを確認するOnPushがあります
- OnPushは、入力の参照変化・イベント・`async`パイプ・`markForCheck()`で更新されます
- Angular 22では、新規Componentの既定がOnPushになり、従来の常時確認は`Eager`に改称されました
- OnPushは不変性を前提とし、Signalと組み合わせると変化が確実に伝わります
- **新規開発では既定のOnPushを活かすのが現在の標準です。状態はSignalで持つのが相性のよい書き方です**

次章では、これまでの変更検知を陰で支えてきたZone.jsと、NgZoneの役割を理解します。
