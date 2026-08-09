---
title: "付録C：エラーカタログ"
---

イベント予約APIで使うProblem Detailsのひな型です。実際のAPIでは、ドメインと回復手順に合わせて追加・削除してください。

## 共通形式

```json
{
  "type": "https://api.example.com/problems/validation-failed",
  "title": "Request validation failed",
  "status": 422,
  "detail": "Two fields are invalid.",
  "instance": "/problems/req_01J4QF",
  "requestId": "req_01J4QF",
  "errors": [
    {
      "pointer": "/capacity",
      "code": "out_of_range",
      "message": "The value must be between 1 and 10000."
    }
  ]
}
```

`type`は問題分類の安定した識別子、`requestId`は個別調査用です。`title`と`detail`の文章でプログラムを分岐させません。

## 要求と入力

| 問題タイプ末尾 | HTTP | 意味 | 同じ要求の再試行 |
|---|---:|---|---|
| `malformed-request` | 400 | JSONやクエリを解釈できない | 不可 |
| `invalid-content-type` | 415 | 本文形式に未対応 | 不可 |
| `not-acceptable` | 406 | 要求表現を生成できない | 不可 |
| `content-too-large` | 413 | ボディが上限を超えた | 不可 |
| `validation-failed` | 422 | 項目の型・形式・範囲が不正 | 不可 |
| `unknown-field` | 422 | 許可しない入力項目がある | 不可 |
| `invalid-query` | 422 | フィルターや並べ替えが不正 | 不可 |
| `cursor-invalid` | 400 | カーソルが不正、条件と不一致 | 最初から取得 |
| `cursor-expired` | 410 | カーソルの有効期限切れ | 最初から取得 |

## 認証と認可

| 問題タイプ末尾 | HTTP | 意味 | 回復 |
|---|---:|---|---|
| `authentication-required` | 401 | 認証情報がない | 認証する |
| `invalid-token` | 401 | 期限切れ・対象違い・検証失敗 | トークンを更新 |
| `insufficient-scope` | 403 | 必要スコープがない | 認可を取り直す |
| `forbidden` | 403 | 対象または操作の権限がない | 管理者へ確認 |
| `step-up-required` | 403 | 追加認証が必要 | 多要素認証を行う |
| `tenant-access-denied` | 404 | テナント境界外。存在を秘匿 | 対象を確認 |

`401`では必要に応じて`WWW-Authenticate`を返します。存在秘匿の方針では、認可失敗を`404`として表すことがあります。

## リソースと状態

| 問題タイプ末尾 | HTTP | 意味 | 回復 |
|---|---:|---|---|
| `event-not-found` | 404 | イベントがない、または閲覧不可 | IDを確認 |
| `reservation-not-found` | 404 | 予約がない、または閲覧不可 | IDを確認 |
| `event-not-published` | 409 | 予約可能な状態ではない | 公開を待つ |
| `event-sold-out` | 409 | 残席不足 | 席数変更・別イベント |
| `invalid-state-transition` | 409 | 現在状態から操作できない | 最新状態を確認 |
| `cancellation-deadline-passed` | 409 | 取消期限を過ぎた | サポートへ確認 |
| `precondition-required` | 428 | 条件付き更新が必要 | `If-Match`を付ける |
| `precondition-failed` | 412 | 指定版が古い | 再取得・統合 |

## 冪等性と競合

| 問題タイプ末尾 | HTTP | 意味 | 回復 |
|---|---:|---|---|
| `idempotency-key-required` | 400 | キー必須操作で欠落 | キーを付ける |
| `idempotency-key-reused` | 409 | 同じキーを異なる内容で使用 | 新しいキーを使う |
| `request-in-progress` | 409 | 同じキーの初回処理中 | 待って同じキーで再送 |
| `duplicate-resource` | 409 | 一意制約に違反 | 既存リソースを取得 |

`request-in-progress`には待ち時間を示せる場合、`Retry-After`を付けます。完了済みの同一要求には保存した成功・失敗応答を返します。

## 制限と一時障害

| 問題タイプ末尾 | HTTP | 意味 | 再試行 |
|---|---:|---|---|
| `rate-limit-exceeded` | 429 | 呼び出し上限超過 | `Retry-After`後 |
| `concurrency-limit-exceeded` | 429 | 同時処理上限超過 | 短いバックオフ後 |
| `service-unavailable` | 503 | 一時的に提供不能 | 制限付きで可 |
| `dependency-unavailable` | 503 | 決済など依存先が利用不能 | 制限付きで可 |
| `gateway-timeout` | 504 | 依存先が時間内に応答しない | 処理状態を確認後 |

再試行では指数バックオフ、ジッター、最大回数、全体締切を使います。非冪等操作には同じ冪等性キーを付けます。

## 非同期とWebhook

| 問題タイプ末尾 | HTTP | 意味 | 回復 |
|---|---:|---|---|
| `job-not-complete` | 409 | 結果がまだ利用できない | 状態を再照会 |
| `job-failed` | 200内の状態 | 非同期ジョブが失敗 | 理由に応じ再作成 |
| `job-expired` | 410 | 結果の保持期限切れ | ジョブを再作成 |
| `webhook-url-not-verified` | 422 | URL所有確認に失敗 | URL・受信設定を確認 |
| `webhook-signature-invalid` | 401 | 署名検証に失敗 | 鍵と生ボディを確認 |

ジョブ状態取得自体が成功しているなら、ジョブの`failed`状態を`200`で返せます。HTTP処理の成功と業務ジョブの成功を分けて考えます。

## カタログ運用

- 問題タイプごとに所有チームを決める
- OpenAPIの応答例と同じ定義を参照する
- 利用者向け文書へ原因と回復手順を書く
- メトリクスは問題タイプを低カーディナリティで集計する
- 型を削除・改名せず、廃止手順を適用する
- 内部例外を新しい公開タイプへ自動的に増殖させない

問題タイプは、エラー文より強い公開契約です。追加する前に、クライアントが異なる行動を取る必要がある分類かを確認してください。

