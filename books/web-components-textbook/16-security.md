---
title: "Shadow DOMはXSSを防がない"
---

Shadow DOMはCSSとDOM探索に境界を作りますが、信頼できない入力を安全なHTMLへ変換する仕組みではありません。内部で`innerHTML`へ外部文字列を渡せば、通常のページと同じようにDOM-based XSSが起こります。

## 文字列はtextContent、構造はDOM APIで組み立てる

利用者が入力したタスク名を表示するだけなら、`textContent`を使います。

```ts
label.textContent = task.label;
```

次のコードは入力をHTMLとして解析します。

```ts
label.innerHTML = task.label;
```

`task.label`に`<img src=x onerror=...>`のような文字列が入ると、意図しない属性や要素がDOMへ作られます。Shadow Root内部でも危険性は変わりません。

リンクを作る場合も、文字列結合よりDOMプロパティを使います。

```ts
const link = document.createElement('a');
link.textContent = task.referenceLabel;
link.href = toSafeUrl(task.referenceUrl);
```

URLは構文とプロトコルを確認します。

```ts
function toSafeUrl(value: string): string {
  const url = new URL(value, document.baseURI);

  if (url.protocol !== 'https:' && url.protocol !== 'http:') {
    throw new TypeError('許可されていないURLです');
  }

  return url.href;
}
```

`javascript:`のような実行可能なURLを弾き、必要なら許可するoriginも絞ります。

## 固定templateと外部入力を混ぜない

作者がソースコードへ固定したtemplate文字列は、外部データを差し込まなければ入力経路になりません。

```ts
const template = document.createElement('template');
template.innerHTML = `
  <button type="button" part="remove-button">削除</button>
`;
```

危険になるのは、文字列補間へ外部入力を入れたときです。

```ts
template.innerHTML = `<p>${task.label}</p>`;
```

構造は固定templateから複製し、値は`textContent`、属性、プロパティで後から設定します。この分離だけで、多くの注入箇所を消せます。

## HTMLを受け取るAPIには明確な理由が要る

リッチテキスト表示のようにHTMLが必要な場合、入力をそのまま注入してはいけません。許可する要素と属性を定めたSanitizerを通します。

新しいHTML Sanitizer APIには、危険な要素や属性を除去して挿入する`Element.setHTML()`があります。ただし、採用時は対象ブラウザの対応状況を確認してください。未対応環境を含む場合は、実績のあるサニタイズライブラリを境界で使います。

```ts
if ('setHTML' in output) {
  output.setHTML(untrustedHtml);
} else {
  output.append(createSanitizedFragment(untrustedHtml));
}
```

`createSanitizedFragment()`はアプリケーションで選んだSanitizerを呼ぶ関数です。「正規表現で`script`タグだけ消す」処理では、属性、URL、SVGなどの別経路を防げません。

## Trusted Typesで危険な代入箇所を制限する

Content Security Policyの`require-trusted-types-for 'script'`を使うと、`innerHTML`などの注入先へ通常の文字列を渡したとき、対応ブラウザが`TypeError`を投げます。

```http
Content-Security-Policy:
  require-trusted-types-for 'script';
  trusted-types task-sanitizer;
```

アプリケーションはSanitizerを通した値だけを`TrustedHTML`として作ります。

```ts
const policy = trustedTypes.createPolicy('task-sanitizer', {
  createHTML(input) {
    return sanitize(input);
  },
});

output.innerHTML = policy.createHTML(untrustedHtml);
```

Trusted Types自体はサニタイズしません。どの変換関数を通ったかを型とCSPで強制する仕組みです。未対応ブラウザも含め、入力を同じSanitizerへ通す設計は維持します。

## Slotへ渡された内容も通常のDOMである

Slotは外部HTMLをShadow DOMへコピーしませんが、Light DOMの内容を表示します。外部から受け取ったHTMLをページ側が安全でない方法で作れば、Slotの有無にかかわらず危険です。

コンポーネント側がSlot内容を`innerHTML`で読み取り、別の場所へ再挿入する処理も避けます。Slotはノードのまま表示し、複製が必要なら信頼境界を確認します。

## closedなShadow Rootへ秘密を置かない

`mode: 'closed'`は`element.shadowRoot`から入口を取得できなくするだけです。認証トークン、個人情報、暗号鍵を隠す保管場所にはなりません。ブラウザへ届いた秘密は、ページで実行されるほかのJavaScriptから完全には守れません。

部品へ必要以上のデータを渡さず、通信資格情報はHttpOnly Cookieなど適切な仕組みで管理します。

## イベントのdetailも外部入力として検証する

Custom Eventはどのスクリプトからでもdispatchできます。

```ts
list.addEventListener('task-toggle', (event) => {
  if (!(event instanceof CustomEvent)) return;

  const { id, completed } = event.detail ?? {};
  if (typeof id !== 'string' || typeof completed !== 'boolean') return;

  updateTask(id, completed);
});
```

イベントが同じページから来たという理由だけで、`detail`の形や権限を信頼しません。サーバー更新では、サーバー側でも認証と認可を行います。

セキュリティの基本は、Shadow DOMという境界へ期待を寄せることではありません。外部入力をHTML、URL、イベント、ネットワークの各入口で検証し、危険なDOM APIを減らすことです。

## 参考資料

- [Cross-site scripting - MDN](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/XSS)
- [Trusted Types API - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API)
- [HTML Sanitizer API - MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_Sanitizer_API)
