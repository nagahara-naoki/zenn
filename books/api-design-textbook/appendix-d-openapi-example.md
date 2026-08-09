---
title: "付録D：OpenAPI完全例"
---

総合演習の一部を、単独で読めるOpenAPI 3.2文書にしました。イベントの一覧・詳細・作成と、冪等な予約作成を含みます。実際のプロジェクトではファイル分割、lint、互換性チェック、契約テストを加えてください。

```yaml
openapi: 3.2.0
info:
  title: Events API
  version: 1.0.0
  summary: イベントの公開と予約を扱うAPI
  description: |
    公開イベントの検索、主催者によるイベント作成、
    参加者による予約作成を提供します。
    エラーはRFC 9457のProblem Details形式で返します。
  contact:
    name: API Support
    url: https://docs.example.com/support
servers:
  - url: https://api.example.com
    description: Production
tags:
  - name: Events
    description: イベントの検索と管理
  - name: Reservations
    description: イベント予約

paths:
  /events:
    get:
      tags: [Events]
      operationId: listEvents
      summary: 公開イベントを検索する
      description: |
        `startsAt`昇順、`id`昇順の安定した順序で返します。
        `cursor`は同じフィルター条件でだけ再利用できます。
      parameters:
        - name: startsAt.gte
          in: query
          description: この日時以降に開始するイベントへ絞る
          schema:
            type: string
            format: date-time
        - name: venueId
          in: query
          schema:
            $ref: '#/components/schemas/VenueId'
        - name: limit
          in: query
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
        - $ref: '#/components/parameters/Cursor'
      responses:
        '200':
          description: 公開イベント一覧
          headers:
            Cache-Control:
              description: 共有キャッシュの規則
              schema:
                type: string
              example: public, max-age=30
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/EventList'
              examples:
                firstPage:
                  value:
                    items:
                      - id: evt_123
                        title: API Design Workshop
                        status: published
                        startsAt: '2026-09-12T04:00:00Z'
                        endsAt: '2026-09-12T07:00:00Z'
                        availability: available
                    page:
                      limit: 20
                      nextCursor: eyJ2IjoxLCJpZCI6ImV2dF8xMjMifQ
                      hasNext: true
        '400':
          $ref: '#/components/responses/BadRequest'
        '422':
          $ref: '#/components/responses/ValidationFailed'
        '429':
          $ref: '#/components/responses/RateLimited'
    post:
      tags: [Events]
      operationId: createEvent
      summary: 下書きイベントを作成する
      security:
        - oauth2: ['events:write']
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateEventInput'
            examples:
              workshop:
                value:
                  title: API Design Workshop
                  startsAt: '2026-09-12T04:00:00Z'
                  endsAt: '2026-09-12T07:00:00Z'
                  venueId: ven_456
                  capacity: 40
      responses:
        '201':
          description: イベントを作成した
          headers:
            Location:
              required: true
              schema:
                type: string
                format: uri-reference
              example: /events/evt_123
            ETag:
              required: true
              schema:
                type: string
              example: '"event-v1"'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/EventDetail'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '422':
          $ref: '#/components/responses/ValidationFailed'

  /events/{eventId}:
    parameters:
      - $ref: '#/components/parameters/EventId'
    get:
      tags: [Events]
      operationId: getEvent
      summary: イベント詳細を取得する
      responses:
        '200':
          description: イベント詳細
          headers:
            ETag:
              required: true
              schema:
                type: string
            Cache-Control:
              required: true
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/EventDetail'
        '304':
          description: 指定したETagから変更されていない
        '404':
          $ref: '#/components/responses/NotFound'
    patch:
      tags: [Events]
      operationId: updateEvent
      summary: イベントを部分更新する
      description: |
        `If-Match`は詳細取得で受け取ったETagです。
        公開後に変更できる項目はタイトルと説明文に限られます。
      security:
        - oauth2: ['events:write']
      parameters:
        - $ref: '#/components/parameters/IfMatch'
      requestBody:
        required: true
        content:
          application/merge-patch+json:
            schema:
              $ref: '#/components/schemas/UpdateEventInput'
      responses:
        '200':
          description: 更新後のイベント
          headers:
            ETag:
              required: true
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/EventDetail'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'
        '412':
          $ref: '#/components/responses/PreconditionFailed'
        '422':
          $ref: '#/components/responses/ValidationFailed'

  /events/{eventId}/reservations:
    post:
      tags: [Reservations]
      operationId: createReservation
      summary: イベントを予約する
      description: |
        同じ要求を再送するときは同じ`Idempotency-Key`を使います。
        キーは24時間保持されます。同じキーと異なる本文は拒否されます。
      security:
        - oauth2: ['reservations:write']
      parameters:
        - $ref: '#/components/parameters/EventId'
        - $ref: '#/components/parameters/IdempotencyKey'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateReservationInput'
            example:
              seats: 2
              paymentMethodId: pm_abc
      responses:
        '201':
          description: 予約を作成した
          headers:
            Location:
              required: true
              schema:
                type: string
                format: uri-reference
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Reservation'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
        '409':
          description: 満席、状態競合、冪等性キーの内容不一致
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
              examples:
                soldOut:
                  value:
                    type: https://api.example.com/problems/event-sold-out
                    title: The event is sold out
                    status: 409
                    detail: No seats remain for this event.
                keyReused:
                  value:
                    type: https://api.example.com/problems/idempotency-key-reused
                    title: The idempotency key was reused
                    status: 409
                    detail: Use a new key for different request content.
        '422':
          $ref: '#/components/responses/ValidationFailed'
        '429':
          $ref: '#/components/responses/RateLimited'
        '503':
          $ref: '#/components/responses/ServiceUnavailable'

components:
  securitySchemes:
    oauth2:
      type: oauth2
      description: Authorization Code + PKCEまたはClient Credentialsを利用する
      flows:
        authorizationCode:
          authorizationUrl: https://auth.example.com/authorize
          tokenUrl: https://auth.example.com/token
          scopes:
            events:write: 所属テナントのイベントを作成・編集する
            reservations:write: 自分の予約を作成・取消する
        clientCredentials:
          tokenUrl: https://auth.example.com/token
          scopes:
            events:write: 所属テナントのイベントを作成・編集する

  parameters:
    EventId:
      name: eventId
      in: path
      required: true
      description: イベントID
      schema:
        $ref: '#/components/schemas/EventId'
    Cursor:
      name: cursor
      in: query
      description: 前の応答で返された不透明なカーソル
      schema:
        type: string
        maxLength: 2048
    IfMatch:
      name: If-Match
      in: header
      required: true
      description: 更新対象の現在のETag
      schema:
        type: string
    IdempotencyKey:
      name: Idempotency-Key
      in: header
      required: true
      description: 再送を同一要求として扱う一意なキー。24時間保持する
      schema:
        type: string
        minLength: 16
        maxLength: 128
        pattern: '^[A-Za-z0-9._~-]+$'

  schemas:
    EventId:
      type: string
      pattern: '^evt_[A-Za-z0-9]+$'
      example: evt_123
    VenueId:
      type: string
      pattern: '^ven_[A-Za-z0-9]+$'
      example: ven_456
    ReservationId:
      type: string
      pattern: '^rsv_[A-Za-z0-9]+$'
      example: rsv_789

    EventStatus:
      type: string
      enum: [draft, scheduled, published, cancelled, completed]

    EventSummary:
      type: object
      additionalProperties: false
      required: [id, title, status, startsAt, endsAt, availability]
      properties:
        id:
          $ref: '#/components/schemas/EventId'
        title:
          type: string
        status:
          const: published
        startsAt:
          type: string
          format: date-time
        endsAt:
          type: string
          format: date-time
        availability:
          type: string
          description: 表示用の目安。予約成功を保証しない
          enum: [available, few_left, sold_out]

    EventDetail:
      type: object
      additionalProperties: false
      required:
        - id
        - title
        - status
        - startsAt
        - endsAt
        - venue
        - capacity
        - createdAt
        - updatedAt
      properties:
        id:
          $ref: '#/components/schemas/EventId'
        title:
          type: string
          minLength: 1
          maxLength: 120
        description:
          type: [string, 'null']
          maxLength: 5000
        status:
          $ref: '#/components/schemas/EventStatus'
        startsAt:
          type: string
          format: date-time
        endsAt:
          type: string
          format: date-time
        venue:
          $ref: '#/components/schemas/VenueSummary'
        capacity:
          type: integer
          minimum: 1
          maximum: 10000
        availableSeats:
          type: integer
          minimum: 0
          description: 予約確約ではなく、応答生成時の値
        createdAt:
          type: string
          format: date-time
        updatedAt:
          type: string
          format: date-time

    VenueSummary:
      type: object
      additionalProperties: false
      required: [id, name]
      properties:
        id:
          $ref: '#/components/schemas/VenueId'
        name:
          type: string

    EventList:
      type: object
      additionalProperties: false
      required: [items, page]
      properties:
        items:
          type: array
          items:
            $ref: '#/components/schemas/EventSummary'
        page:
          $ref: '#/components/schemas/CursorPage'

    CursorPage:
      type: object
      additionalProperties: false
      required: [limit, hasNext]
      properties:
        limit:
          type: integer
          minimum: 1
          maximum: 100
        nextCursor:
          type: [string, 'null']
        hasNext:
          type: boolean

    CreateEventInput:
      type: object
      additionalProperties: false
      required: [title, startsAt, endsAt, venueId, capacity]
      properties:
        title:
          type: string
          minLength: 1
          maxLength: 120
        description:
          type: string
          maxLength: 5000
        startsAt:
          type: string
          format: date-time
        endsAt:
          type: string
          format: date-time
        venueId:
          $ref: '#/components/schemas/VenueId'
        capacity:
          type: integer
          minimum: 1
          maximum: 10000

    UpdateEventInput:
      type: object
      additionalProperties: false
      minProperties: 1
      properties:
        title:
          type: string
          minLength: 1
          maxLength: 120
        description:
          type: [string, 'null']
          maxLength: 5000
        startsAt:
          type: string
          format: date-time
        endsAt:
          type: string
          format: date-time
        capacity:
          type: integer
          minimum: 1
          maximum: 10000

    CreateReservationInput:
      type: object
      additionalProperties: false
      required: [seats, paymentMethodId]
      properties:
        seats:
          type: integer
          minimum: 1
          maximum: 10
        paymentMethodId:
          type: string
          minLength: 1
          maxLength: 128

    Reservation:
      type: object
      additionalProperties: false
      required: [id, eventId, status, seats, createdAt]
      properties:
        id:
          $ref: '#/components/schemas/ReservationId'
        eventId:
          $ref: '#/components/schemas/EventId'
        status:
          type: string
          enum: [pending, confirmed, failed, expired, cancelled, attended]
        seats:
          type: integer
          minimum: 1
          maximum: 10
        createdAt:
          type: string
          format: date-time

    InvalidParameter:
      type: object
      additionalProperties: false
      required: [code, message]
      properties:
        pointer:
          type: string
          description: JSON Pointer。ボディ外の入力では省略する
        location:
          type: string
          enum: [path, query, header, body]
        name:
          type: string
        code:
          type: string
        message:
          type: string

    Problem:
      type: object
      required: [type, title, status]
      properties:
        type:
          type: string
          format: uri-reference
        title:
          type: string
        status:
          type: integer
          minimum: 100
          maximum: 599
        detail:
          type: string
        instance:
          type: string
          format: uri-reference
        requestId:
          type: string
        errors:
          type: array
          items:
            $ref: '#/components/schemas/InvalidParameter'

  responses:
    BadRequest:
      description: 要求を解釈できない
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
    Unauthorized:
      description: 有効な認証情報がない
      headers:
        WWW-Authenticate:
          required: true
          schema:
            type: string
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
          example:
            type: https://api.example.com/problems/authentication-required
            title: Authentication is required
            status: 401
    Forbidden:
      description: 操作する権限がない
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
    NotFound:
      description: 対象がない、または閲覧できない
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
    ValidationFailed:
      description: 入力が制約を満たさない
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
          example:
            type: https://api.example.com/problems/validation-failed
            title: Request validation failed
            status: 422
            errors:
              - pointer: /capacity
                location: body
                code: out_of_range
                message: The value must be between 1 and 10000.
    PreconditionFailed:
      description: 対象が指定された版から変更されている
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
          example:
            type: https://api.example.com/problems/precondition-failed
            title: The resource has changed
            status: 412
    RateLimited:
      description: 利用上限を超えた
      headers:
        Retry-After:
          required: true
          schema:
            type: integer
            minimum: 1
          example: 30
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
    ServiceUnavailable:
      description: 一時的に処理できない
      headers:
        Retry-After:
          schema:
            type: integer
            minimum: 1
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
```

## この例で省略したもの

完全例という題名ですが、製品の全要件を一ファイルへ詰めたという意味ではありません。次はプロジェクト側で追加します。

- イベント公開・中止、予約取消、Webhook、エクスポートの全操作
- OAuth認可サーバー固有の要件とトークン形式
- 業務ルールを説明する外部ドキュメントへのリンク
- 複数のエラー例、Webhookペイロード例
- API版、非推奨、利用規約、サービス水準の説明
- lint規則、破壊的変更の判定基準、SDK生成設定

OpenAPIが大きくなったら、パス、スキーマ、共通応答を複数ファイルへ分けても構いません。配布時には参照を解決した単一文書も生成し、利用ツールが同じ契約を読めるようにします。
