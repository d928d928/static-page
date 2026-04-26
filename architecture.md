# BookFellows インフラ構成図

```mermaid
graph TB
    subgraph Client["👤 クライアント"]
        Browser["ブラウザ / スマートフォン"]
    end

    subgraph Vercel["▲ Vercel"]
        NextJS["Next.js\nフロントエンド"]
    end

    subgraph Render["🟣 Render"]
        Django["Django REST API\n(Gunicorn)"]
        PG[("PostgreSQL")]
    end

    subgraph CF["☁️ Cloudflare"]
        R2["R2\nメディアストレージ"]
    end

    subgraph AWS["☁️ AWS"]
        SES["SES\nメール配信"]
        SQS["SQS\nbookfellows-push"]
        DLQ["SQS DLQ\n(3回失敗で退避)"]
        Lambda["Lambda\npush-worker"]
    end

    subgraph External["🌐 外部サービス"]
        Stripe["Stripe\n決済"]
        GoogleOAuth["Google OAuth"]
        WebPushAPI["Web Push API\n(FCM / APNS)"]
    end

    subgraph CICD["⚙️ GitHub / CI CD"]
        GH["GitHub\nリポジトリ"]
        Actions["GitHub Actions"]
    end

    Browser -->|"HTTPS"| NextJS
    NextJS -->|"API /api/v1/*\n(rewrite proxy)"| Django
    Django <-->|"ORM"| PG
    Django -->|"画像アップロード"| R2
    Browser -->|"メディア直接取得"| R2
    Django -->|"トランザクションメール"| SES
    Django -->|"決済 API"| Stripe
    Browser -->|"OAuth フロー"| GoogleOAuth
    GoogleOAuth -->|"トークン検証"| Django

    Django -->|"① イベント発行"| SQS
    SQS -->|"失敗時"| DLQ
    SQS -->|"② トリガー"| Lambda
    Lambda -->|"③ webpush 送信"| WebPushAPI
    WebPushAPI -->|"④ プッシュ通知"| Browser
    Lambda -->|"⑤ スタレSub削除"| Django

    GH -->|"push"| Actions
    Actions -->|"deploy"| NextJS
    Actions -->|"deploy hook"| Django
    Actions -->|"update-function-code"| Lambda

    classDef aws fill:#FF9900,stroke:#FF9900,color:#000
    classDef vercel fill:#000,stroke:#000,color:#fff
    classDef render fill:#6C47FF,stroke:#6C47FF,color:#fff
    classDef cf fill:#F38020,stroke:#F38020,color:#fff
    classDef ext fill:#4A90D9,stroke:#4A90D9,color:#fff
    classDef cicd fill:#24292E,stroke:#24292E,color:#fff

    class SQS,DLQ,Lambda,SES aws
    class NextJS vercel
    class Django,PG render
    class R2 cf
    class Stripe,GoogleOAuth,WebPushAPI ext
    class GH,Actions cicd
```

## フローの補足

| フロー | 同期 / 非同期 |
|---|---|
| ブラウザ → Vercel → Django | 同期 |
| Django → Cloudflare R2 | 同期 |
| Django → AWS SES / Stripe | 同期 |
| Django → SQS → Lambda → Web Push | 非同期 |
| GitHub Actions → Vercel / Render / Lambda | 非同期（CD） |
