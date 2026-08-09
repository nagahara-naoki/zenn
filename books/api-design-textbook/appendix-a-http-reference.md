---
title: "付録A：HTTP早見表"
---

設計レビューで参照しやすいよう、本書で使ったHTTP要素をまとめます。表は選択の出発点です。最終判断では、各メソッドとステータスの意味を[RFC 9110](https://www.rfc-editor.org/rfc/rfc9110)で確認してください。

## HTTPメソッド

| メソッド | 安全 | 冪等 | 主な用途 | 成功応答の例 |
|---|:---:|:---:|---|---|
| `GET` | ○ | ○ | 表現を取得 | `200`、`304` |
| `HEAD` | ○ | ○ | 本文なしでメタデータを取得 | `200`、`304` |
| `POST` | × | × | 作成、業務操作、処理受付 | `200`、`201`、`202`、`204` |
| `PUT` | × | ○ | 指定URIの表現を全置換 | `200`、`201`、`204` |
| `PATCH` | × | 記法次第 | 部分更新 | `200`、`204` |
| `DELETE` | × | ○ | リソース削除 | `202`、`204` |
| `OPTIONS` | ○ | ○ | 通信選択肢を取得 | `200`、`204` |
| `QUERY` | ○ | ○ | 内容を伴う問い合わせ | `200` |

安全は「状態変更の意図がない」、冪等は「同じ要求を複数回送った効果が一回と同じ」という意味です。実装が監査ログを残すことまで禁止する性質ではありません。

## 成功とリダイレクト

| コード | 意味 | 使いどころ |
|---:|---|---|
| `200 OK` | 成功し、本文を返す | 取得、更新結果、同期操作 |
| `201 Created` | リソースを作成 | `Location`と作成表現を返す |
| `202 Accepted` | 処理を受け付けたが未完了 | 状態照会先を返す |
| `204 No Content` | 成功し、本文はない | 更新・削除で表現不要 |
| `206 Partial Content` | 範囲要求の一部 | バイト範囲ダウンロード |
| `304 Not Modified` | キャッシュ済み表現を再利用可能 | `If-None-Match`への応答 |

`202`は将来の成功を保証しません。ジョブの失敗状態も設計します。`204`には本文を付けません。

## クライアント側のエラー

| コード | 代表的な状況 | 主な回復 |
|---:|---|---|
| `400 Bad Request` | JSON構文不正、要求を解釈できない | 要求を修正 |
| `401 Unauthorized` | 認証なし、トークン無効 | 再認証 |
| `403 Forbidden` | 認証済みだが権限不足 | 権限を確認 |
| `404 Not Found` | 存在しない、または存在を秘匿 | ID・権限を確認 |
| `405 Method Not Allowed` | 対応しないメソッド | `Allow`を確認 |
| `406 Not Acceptable` | 要求された表現を生成できない | `Accept`を修正 |
| `409 Conflict` | 現在状態との競合、重複 | 状態を確認 |
| `410 Gone` | 恒久的に削除済み | 代替へ移行 |
| `412 Precondition Failed` | `If-Match`などが不成立 | 最新版を再取得 |
| `413 Content Too Large` | ボディが上限超過 | 小さくする |
| `415 Unsupported Media Type` | `Content-Type`非対応 | 形式を変更 |
| `422 Unprocessable Content` | 構文は正しいが内容不正 | 指摘項目を修正 |
| `429 Too Many Requests` | 利用上限超過 | `Retry-After`後に再試行 |

## サーバー側のエラー

| コード | 状況 | 再試行の目安 |
|---:|---|---|
| `500 Internal Server Error` | 予期しない内部エラー | 冪等性を確認し、制限付きで |
| `502 Bad Gateway` | 上流から不正な応答 | 一時的ならバックオフ |
| `503 Service Unavailable` | 過負荷、保守、一時障害 | `Retry-After`を尊重 |
| `504 Gateway Timeout` | 上流が時間内に応答しない | 処理済みか確認してから |

`5xx`だから無条件で再試行するのではなく、操作の冪等性、試行上限、全体締切を確認します。

## よく使うリクエストヘッダー

| ヘッダー | 用途 | 例 |
|---|---|---|
| `Authorization` | 認証情報 | `Bearer eyJ...` |
| `Accept` | 受け取りたい表現 | `application/json` |
| `Content-Type` | 送信本文の種類 | `application/json` |
| `If-Match` | 現在版が一致する場合だけ更新 | `"event-v7"` |
| `If-None-Match` | 版が違う場合だけ本文を取得 | `"event-v7"` |
| `Idempotency-Key` | 非冪等操作の再送を重複排除 | ランダムで一意な値 |
| `traceparent` | 分散トレースを伝播 | W3C形式 |

`Idempotency-Key`は2026年8月時点でIETFドラフトです。採用APIが意味を定義します。

## よく使うレスポンスヘッダー

| ヘッダー | 用途 | 例 |
|---|---|---|
| `Location` | 作成物・状態照会先 | `/events/evt_123` |
| `ETag` | 表現または版の識別子 | `"event-v8"` |
| `Cache-Control` | 保存と再利用の規則 | `private, no-cache` |
| `Vary` | 表現が変わる要求ヘッダー | `Accept-Encoding` |
| `Retry-After` | 再試行までの時間・日時 | `30` |
| `WWW-Authenticate` | 認証方法と失敗情報 | `Bearer realm="api"` |
| `Deprecation` | 非推奨となった時点 | `@1798761600` |
| `Sunset` | 提供終了予定 | HTTP-date |
| `Link` | 関連文書や次ページ | `<...>; rel="next"` |

## Cache-Control指示

| 指示 | 意味 |
|---|---|
| `public` | 共有キャッシュへ保存可能 |
| `private` | 利用者専用キャッシュだけ保存可能 |
| `no-store` | 保存しない |
| `no-cache` | 再利用前に検証する |
| `max-age=N` | N秒の鮮度 |
| `s-maxage=N` | 共有キャッシュ向けの鮮度 |
| `must-revalidate` | 古くなった応答を検証なしで使わない |

## メディアタイプ

| メディアタイプ | 用途 |
|---|---|
| `application/json` | 通常のJSON表現 |
| `application/problem+json` | Problem Details |
| `application/merge-patch+json` | JSON Merge Patch |
| `application/json-patch+json` | JSON Patch |
| `text/csv` | CSVエクスポート |

参考: [HTTP Semantics（RFC 9110）](https://www.rfc-editor.org/rfc/rfc9110)、[HTTP Caching（RFC 9111）](https://www.rfc-editor.org/rfc/rfc9111)、[QUERY Method（RFC 10008）](https://www.rfc-editor.org/rfc/rfc10008)

