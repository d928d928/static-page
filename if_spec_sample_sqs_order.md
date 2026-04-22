# インタフェース仕様書 受注情報連携 (Service A → Middleware)

**文書ID**：IF-A-MQ-001
**連携方式**：Amazon SQS (メッセージキュー)
**連携方向**：Service A → Middleware

---

## 改版履歴

| 版数 | 日付       | 改版内容       | 作成者  | 承認者       |
| ---- | ---------- | -------------- | ------- | ------------ |
| 0.1  | 2026-04-22 | 初版作成       | Kuniya  | -            |
| 0.2  | 2026-04-25 | レビュー反映   | Kuniya  | -            |

---

## 1. 連携概要

### 1.1 業務概要

| 項目             | 内容                                                  |
| ---------------- | ----------------------------------------------------- |
| 連携名           | 受注情報連携                                          |
| 業務概要         | Service Aで登録された受注情報をMiddleware経由でService Bへ連携する |
| トリガー         | Service A上で受注が確定したタイミング                 |
| 連携タイミング   | 準リアルタイム（3秒以内にキューへ投入）               |
| 想定件数         | 平常時：平均50件/分 / ピーク時：最大500件/分          |
| 業務影響         | 連携失敗時は受注処理が止まるため、24時間監視対象      |
| 関連業務フロー   | `業務フロー_受注情報連携_v1.0.pptx`                   |

### 1.2 連携シーケンス概要

```
[Service A]
    │ (1) 受注確定
    │ (2) メッセージ送信 (a-to-middleware キュー)
    ▼
[SQS: a-to-middleware]
    │ (3) ポーリング
    ▼
[Middleware (ECS Fargate)]
    │ (4) バリデーション
    │ (5) Outbox登録 + DB更新 (トランザクション)
    │ (6) Outboxリレイ → b-to-middleware-from-m キュー
    │ (7) 処理結果を response-to-a キューへ送信
    ▼
[SQS: response-to-a]
    │ (8) Service Aがポーリング
    ▼
[Service A] ← 処理結果通知
```

### 1.3 使用キュー一覧

| キュー名                   | 方向       | 所有アカウント | 用途                              |
| -------------------------- | ---------- | -------------- | --------------------------------- |
| `a-to-middleware`          | A → M      | Middleware     | 受注情報の送信用                  |
| `a-to-middleware-dlq`      | -          | Middleware     | 上記キューのDLQ                   |
| `response-to-a`            | M → A      | Middleware     | 処理結果の応答用                  |
| `response-to-a-dlq`        | -          | Middleware     | 上記キューのDLQ                   |

---

## 2. キュー仕様

### 2.1 `a-to-middleware` キュー (Service A → Middleware)

| 項目                     | 設定値                                                          |
| ------------------------ | --------------------------------------------------------------- |
| キュー名                 | `a-to-middleware`                                               |
| キューURL                | `https://sqs.ap-northeast-1.amazonaws.com/<Middleware_Account>/a-to-middleware` |
| キューARN                | `arn:aws:sqs:ap-northeast-1:<Middleware_Account>:a-to-middleware` |
| リージョン               | ap-northeast-1 (東京)                                           |
| キュータイプ             | 標準キュー                                                      |
| 配信保証                 | At-Least-Once（重複は冪等性キーで排除）                         |
| 順序保証                 | なし（業務上順序は不問。受信順で処理可）                        |
| メッセージ保持期間       | 4日 (345600秒)                                                  |
| 可視性タイムアウト       | 60秒                                                            |
| 受信待機時間 (Long Poll) | 20秒                                                            |
| 最大メッセージサイズ     | 256 KB                                                          |
| 配信遅延                 | 0秒                                                             |
| 暗号化                   | SSE-KMS（カスタマーマネージドCMK）                              |
| KMSキーARN               | `arn:aws:kms:ap-northeast-1:<Middleware_Account>:key/<key-id>`  |
| DLQ                      | `a-to-middleware-dlq`                                           |
| maxReceiveCount          | 5                                                               |

### 2.2 `response-to-a` キュー (Middleware → Service A)

| 項目                     | 設定値                                                          |
| ------------------------ | --------------------------------------------------------------- |
| キュー名                 | `response-to-a`                                                 |
| キューARN                | `arn:aws:sqs:ap-northeast-1:<Middleware_Account>:response-to-a` |
| キュータイプ             | 標準キュー                                                      |
| 配信保証                 | At-Least-Once                                                   |
| 順序保証                 | なし                                                            |
| メッセージ保持期間       | 4日                                                             |
| 可視性タイムアウト       | 30秒                                                            |
| 最大メッセージサイズ     | 256 KB                                                          |
| 暗号化                   | SSE-KMS（カスタマーマネージドCMK）                              |
| DLQ                      | `response-to-a-dlq`                                             |
| maxReceiveCount          | 5                                                               |

### 2.3 DLQ仕様

| 項目                   | `a-to-middleware-dlq`           | `response-to-a-dlq`             |
| ---------------------- | ------------------------------- | ------------------------------- |
| メッセージ保持期間     | 14日                            | 14日                            |
| 監視                   | Middleware側で滞留監視          | Middleware側で滞留監視          |
| 再処理                 | Middlewareがリプレイ            | Middlewareがリプレイ            |
| 通知先                 | Slack `#middleware-alerts`      | Slack `#middleware-alerts`      |

---

## 3. アクセス制御

### 3.1 Service A側で必要なIAM権限

以下の権限をService A側のIAMロール（`ServiceA-Producer-Role` / `ServiceA-Consumer-Role`）に付与する。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SendMessageToMiddleware",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:GetQueueUrl",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:ap-northeast-1:<Middleware_Account>:a-to-middleware"
    },
    {
      "Sid": "ReceiveMessageFromMiddleware",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:ChangeMessageVisibility",
        "sqs:GetQueueUrl",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:ap-northeast-1:<Middleware_Account>:response-to-a"
    },
    {
      "Sid": "UseMiddlewareKMSKey",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:ap-northeast-1:<Middleware_Account>:key/<key-id>"
    }
  ]
}
```

### 3.2 キューリソースポリシー (Middleware側で設定)

`a-to-middleware` キュー側に設定するポリシー（送信許可）：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowServiceASend",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ServiceA_Account>:role/ServiceA-Producer-Role"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:ap-northeast-1:<Middleware_Account>:a-to-middleware",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "<ServiceA_Account>"
        }
      }
    }
  ]
}
```

### 3.3 ネットワーク要件

| 項目                       | 内容                                                          |
| -------------------------- | ------------------------------------------------------------- |
| Service A側に必要なエンドポイント | `com.amazonaws.ap-northeast-1.sqs` (Interface VPC Endpoint) |
|                            | `com.amazonaws.ap-northeast-1.kms` (Interface VPC Endpoint)   |
| Middleware側に必要なエンドポイント | 同上                                                  |
| VPC間接続                  | 不要（VPC Peering / PrivateLinkは使用しない）                 |
| インターネット経由         | なし（AWSバックボーン内で完結）                               |

---

## 4. メッセージ属性 (Message Attributes)

SQSメッセージ属性として以下を必ず設定する。

| 属性名              | データ型    | 必須/任意 | 説明                                           |
| ------------------- | ----------- | --------- | ---------------------------------------------- |
| `messageType`       | String      | 必須      | `OrderCreate` / `OrderUpdate` / `OrderCancel`  |
| `messageVersion`    | String      | 必須      | スキーマバージョン（例：`1.0`）                |
| `correlationId`     | String      | 必須      | 分散トレース用の相関ID (UUID v4)               |
| `idempotencyKey`    | String      | 必須      | 冪等性キー (UUID v4)                           |
| `sourceSystem`      | String      | 必須      | `ServiceA` 固定                                |
| `sentAt`            | String      | 必須      | ISO 8601形式の送信日時                         |

### 4.1 メッセージ属性サンプル

```
messageType:     OrderCreate
messageVersion:  1.0
correlationId:   550e8400-e29b-41d4-a716-446655440000
idempotencyKey:  7c9e6679-7425-40de-944b-e07fc1f90ae7
sourceSystem:    ServiceA
sentAt:          2026-04-22T10:00:00+09:00
```

---

## 5. メッセージ本文仕様

### 5.1 フォーマット

| 項目         | 内容                  |
| ------------ | --------------------- |
| フォーマット | JSON                  |
| 文字コード   | UTF-8（BOMなし）      |
| 改行コード   | LF (`\n`)             |
| 圧縮         | なし                  |

### 5.2 リクエストメッセージ構造 (`OrderCreate`)

| 階層  | フィールド日本語名   | 物理名            | データ型              | 繰返    | 必須/任意       | 説明・制約                            |
| ----- | -------------------- | ----------------- | --------------------- | ------- | --------------- | ------------------------------------- |
| 1     | メッセージID         | messageId         | String(36)            | 1       | 必須            | UUID v4、本メッセージの一意識別子     |
| 2     | 受注ヘッダー         | orderHeader       | Object                | 1       | 必須            |                                       |
| 2.1   | 受注番号             | orderNo           | String(20)            | 1       | 必須            | 英数字、A側で採番                     |
| 2.2   | 受注日               | orderDate         | Date (YYYY-MM-DD)     | 1       | 必須            | JST                                   |
| 2.3   | 顧客コード           | customerCode      | String(10)            | 1       | 必須            |                                       |
| 2.4   | 納期                 | deliveryDate      | Date (YYYY-MM-DD)     | 1       | 任意            | JST                                   |
| 2.5   | 合計金額             | totalAmount       | Decimal(12,2)         | 1       | 必須            | 円単位、税込                          |
| 2.6   | 通貨                 | currency          | Enum (JPY/USD)        | 1       | 必須            |                                       |
| 2.7   | 備考                 | remarks           | String(200)           | 1       | 任意            |                                       |
| 3     | 明細                 | items             | Array<Object>         | 1..100  | 必須            | 最大100明細                           |
| 3.1   | 行番号               | lineNo            | Integer               | 1       | 必須            | 1以上の整数、明細内で一意             |
| 3.2   | 商品コード           | itemCode          | String(20)            | 1       | 必須            |                                       |
| 3.3   | 商品名               | itemName          | String(100)           | 1       | 必須            |                                       |
| 3.4   | 数量                 | quantity          | Integer               | 1       | 必須            | 1以上                                 |
| 3.5   | 単価                 | unitPrice         | Decimal(10,2)         | 1       | 必須            | 0以上                                 |
| 3.6   | 小計                 | subtotal          | Decimal(12,2)         | 1       | 必須            | quantity × unitPrice と一致すること   |

### 5.3 リクエストメッセージサンプル (`OrderCreate`)

```json
{
  "messageId": "msg-550e8400-e29b-41d4-a716-446655440000",
  "orderHeader": {
    "orderNo": "ORD-20260422-0001",
    "orderDate": "2026-04-22",
    "customerCode": "CUST-A0001",
    "deliveryDate": "2026-04-30",
    "totalAmount": 16500.00,
    "currency": "JPY",
    "remarks": "至急対応希望"
  },
  "items": [
    {
      "lineNo": 1,
      "itemCode": "ITEM-A001",
      "itemName": "商品A",
      "quantity": 10,
      "unitPrice": 1500.00,
      "subtotal": 15000.00
    },
    {
      "lineNo": 2,
      "itemCode": "ITEM-B002",
      "itemName": "商品B",
      "quantity": 5,
      "unitPrice": 300.00,
      "subtotal": 1500.00
    }
  ]
}
```

### 5.4 `OrderUpdate` / `OrderCancel` メッセージ

- `OrderUpdate`：`OrderCreate` と同じスキーマ。`orderNo` で対象を特定
- `OrderCancel`：`messageId` + `orderHeader.orderNo` のみ必須。他項目は任意

---

## 6. レスポンスメッセージ仕様 (`response-to-a`)

### 6.1 レスポンスメッセージ属性

| 属性名              | データ型 | 必須/任意 | 説明                                          |
| ------------------- | -------- | --------- | --------------------------------------------- |
| `messageType`       | String   | 必須      | `OrderCreateResult` 等、リクエスト種別に対応  |
| `messageVersion`    | String   | 必須      | スキーマバージョン                            |
| `correlationId`     | String   | 必須      | リクエスト時の `correlationId` をそのまま返却 |
| `originalIdempotencyKey` | String | 必須   | リクエスト時の `idempotencyKey`               |
| `sourceSystem`      | String   | 必須      | `Middleware` 固定                             |
| `sentAt`            | String   | 必須      | ISO 8601形式                                  |

### 6.2 レスポンスメッセージ構造

| 階層  | フィールド日本語名       | 物理名            | データ型                  | 繰返    | 必須/任意       | 説明                                    |
| ----- | ------------------------ | ----------------- | ------------------------- | ------- | --------------- | --------------------------------------- |
| 1     | メッセージID             | messageId         | String(36)                | 1       | 必須            | UUID v4                                 |
| 2     | 元リクエストID           | originalMessageId | String(36)                | 1       | 必須            | リクエストの `messageId`                |
| 3     | ステータス               | status            | Enum (SUCCESS/FAILURE)    | 1       | 必須            |                                         |
| 4     | 受注番号                 | orderNo           | String(20)                | 1       | 必須            | 処理対象の受注番号                      |
| 5     | 中間ID                   | middlewareOrderId | String(30)                | 1       | 条件付き必須    | 正常時のみ必須。Middlewareで採番        |
| 6     | 処理日時                 | processedAt       | DateTime (ISO8601)        | 1       | 必須            | 処理完了日時                            |
| 7     | エラー情報               | error             | Object                    | 1       | 条件付き必須    | `status=FAILURE` の場合のみ必須         |
| 7.1   | エラーコード             | errorCode         | String(10)                | 1       | 必須            | 例：`E4001`                             |
| 7.2   | エラーメッセージ         | errorMessage      | String(500)               | 1       | 必須            |                                         |
| 7.3   | 詳細エラー               | details           | Array<Object>             | 0..N    | 任意            |                                         |
| 7.3.1 | フィールド名             | field             | String(100)               | 1       | 必須            |                                         |
| 7.3.2 | エラー理由               | reason            | String(200)               | 1       | 必須            |                                         |

### 6.3 レスポンスサンプル (正常系)

```json
{
  "messageId": "msg-6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "originalMessageId": "msg-550e8400-e29b-41d4-a716-446655440000",
  "status": "SUCCESS",
  "orderNo": "ORD-20260422-0001",
  "middlewareOrderId": "MW-ORD-20260422-000001",
  "processedAt": "2026-04-22T10:00:03+09:00"
}
```

### 6.4 レスポンスサンプル (異常系)

```json
{
  "messageId": "msg-6ba7b810-9dad-11d1-80b4-00c04fd430c9",
  "originalMessageId": "msg-550e8400-e29b-41d4-a716-446655440000",
  "status": "FAILURE",
  "orderNo": "ORD-20260422-0001",
  "processedAt": "2026-04-22T10:00:03+09:00",
  "error": {
    "errorCode": "E4001",
    "errorMessage": "バリデーションエラーが発生しました",
    "details": [
      { "field": "orderHeader.totalAmount", "reason": "明細の小計合計と一致しません" },
      { "field": "items[0].quantity", "reason": "1以上の整数を指定してください" }
    ]
  }
}
```

---

## 7. エラーコード一覧

| エラーコード | 分類               | 説明                             | Service A側の対処                  |
| ------------ | ------------------ | -------------------------------- | ---------------------------------- |
| E4001        | バリデーション     | リクエスト内容の不整合           | 内容を修正して再送                 |
| E4002        | 業務ルール違反     | 業務エラー (例：納期 < 受注日)   | 業務部門と調整後に再送             |
| E4090        | 重複エラー         | 同一 `orderNo` が処理済み        | 既処理の確認。不要なら無視         |
| E5001        | システムエラー     | Middleware内部エラー             | 自動リトライ。継続時は連絡         |
| E5002        | 下流システムエラー | Service B側のエラー              | Middlewareが自動リトライ           |
| E5030        | メンテナンス中     | Middleware計画停止中             | メンテナンス終了後に自動再処理     |

---

## 8. 処理ルール

### 8.1 冪等性

- `idempotencyKey` で24時間以内の重複を排除
- 同一キーで再送された場合、初回の処理結果を返却（再処理は行わない）
- キーはUUID v4形式、Service A側で生成

### 8.2 リトライポリシー

| 主体         | 対象                          | 方式                                    |
| ------------ | ----------------------------- | --------------------------------------- |
| SQS          | 可視性タイムアウト経過後      | 最大5回（maxReceiveCount=5）            |
| Middleware   | 下流システムエラー (E5002)    | 指数バックオフ、最大3回                 |
| Service A    | 応答メッセージ未着            | タイムアウト60秒後に問い合わせAPIで確認 |

### 8.3 DLQ運用

- DLQに移送されたメッセージは、**原則Middleware側で原因調査・再処理**を実施
- 業務判断が必要な場合、Service A側へ連絡（連絡体制は別紙参照）
- DLQ滞留1件で即時アラート、10件超過で重大度引き上げ

### 8.4 メッセージ順序

- 標準キューのため**順序保証なし**
- 業務上、同一 `orderNo` に対して `Create` → `Update` → `Cancel` の順で処理される必要がある場合、Service A側で**前のメッセージの応答を受けてから次を送る**運用とする
- または、Middleware側で `orderNo` 単位のシーケンス番号管理を検討（別途協議）

### 8.5 メッセージサイズ超過時

- 256KB上限を超える場合、**S3 Claim Checkパターン**を採用
- メッセージ本文には `s3Reference` フィールドを含め、本体はS3に配置
- 詳細は別途仕様化（本版では対象外）

---

## 9. 非機能要件

| 項目                   | 要件                                                 |
| ---------------------- | ---------------------------------------------------- |
| スループット (平常)    | 50 msg/分 = 約1 msg/秒                               |
| スループット (ピーク)  | 500 msg/分 = 約8 msg/秒                              |
| 処理完了目安時間       | 95%のメッセージを5秒以内に処理完了                   |
| 可用性                 | 99.9%（SQS・Middleware合算）                         |
| メンテナンス窓         | 毎月第2日曜 2:00-6:00 JST                            |
| メンテナンス時の挙動   | Middleware停止中もSQSへの投入は可能（キューで滞留）  |
| キュー滞留許容         | 通常10件以下、30分間100件超過で警告                  |

---

## 10. セキュリティ要件

| 項目               | 内容                                                    |
| ------------------ | ------------------------------------------------------- |
| 通信路暗号化       | AWS SDKによるTLS 1.2以上                                |
| 保存時暗号化       | SSE-KMS（カスタマーマネージドCMK）                      |
| 認証               | IAMロール（クロスアカウントAssumeRole）                 |
| 認可               | IAMポリシー + キューリソースポリシー + KMSキーポリシー  |
| 個人情報の扱い     | 顧客コードのみ含む（氏名・住所等は含まない）            |
| 監査ログ           | CloudTrailにて送受信操作を記録（7年保持）               |

---

## 11. 運用要件

### 11.1 監視項目

| 指標                              | 閾値              | 通知先                     |
| --------------------------------- | ----------------- | -------------------------- |
| キュー滞留件数                    | 100件超過（30分） | Slack `#middleware-alerts` |
| DLQ滞留件数                       | 1件以上           | Slack + PagerDuty          |
| メッセージ処理エラー率            | 5%超過（5分）     | Slack `#middleware-alerts` |
| 処理レイテンシ (p95)              | 10秒超過          | Slack `#middleware-alerts` |
| KMS Decrypt APIエラー             | 1件以上           | Slack + PagerDuty          |

### 11.2 連絡体制

| 時間帯           | 連絡先              | 備考                   |
| ---------------- | ------------------- | ---------------------- |
| 平日 9:00-18:00  | 運用チーム (一次)   | Slack DM → 電話        |
| 夜間・休日       | オンコール担当      | PagerDuty              |
| Service A側連絡先 | (別途共有)         |                        |

---

## 12. 関連ドキュメント

| ドキュメント名            | 版数 | 格納先                   |
| ------------------------- | ---- | ------------------------ |
| 業務フロー図_受注情報連携 | 1.0  | `docs/business-flow/`    |
| 認証・ネットワーク設計書  | 1.0  | `docs/infra/`            |
| 運用手順書                | 1.0  | `docs/operations/`       |
| テスト計画書              | -    | `docs/test/`             |

---

## 13. 未決事項・課題

| No | 内容                                                 | 担当   | 期限       | 状態   |
| -- | ---------------------------------------------------- | ------ | ---------- | ------ |
| 1  | ピーク500 msg/分の妥当性確認（Service A側の実測値）  | A側    | 2026-05-10 | 調整中 |
| 2  | 順序保証の業務要件有無の最終確認                     | A側    | 2026-05-10 | 調整中 |
| 3  | 応答メッセージ未着時のSLA合意                        | 合同   | 2026-05-15 | 未着手 |
| 4  | S3 Claim Checkパターン仕様の確定                     | M側    | 2026-05-20 | 未着手 |

---

## 付録A：命名規約

| 要素               | 規約                                                  | 例                          |
| ------------------ | ----------------------------------------------------- | --------------------------- |
| キュー名           | `<source>-to-<target>` / `response-to-<target>`       | `a-to-middleware`           |
| DLQ名              | `<queue_name>-dlq`                                    | `a-to-middleware-dlq`       |
| messageId          | `msg-` + UUID v4                                      | `msg-550e8400-...`          |
| Middleware採番ID   | `MW-<entity>-YYYYMMDD-NNNNNN`                         | `MW-ORD-20260422-000001`    |
