# ITヘルプデスク AIチャットボット アーキテクチャ図

**フェーズ**: POC  
**作成日**: 2026-04-07

---

## 1. システム全体アーキテクチャ

```mermaid
graph TB
    %% ── ユーザー ──
    User(["👤 一般社員\nブラウザ"])

    %% ── AWS Edge ──
    subgraph EDGE["🌐 AWS グローバルエッジ"]
        CF["☁️ Amazon CloudFront\nCDN・HTTPS終端"]
        WAF["🛡️ AWS WAF\nWeb Application Firewall"]
    end

    %% ── VPC ──
    subgraph VPC["🔒 Amazon VPC"]

        subgraph PUB["Public Subnet"]
            APIGW["🔀 Amazon API Gateway\nHTTP API・JWT認証・レート制限"]
        end

        subgraph PRIV_APP["Private Subnet（Application）"]
            ECS["🐳 Amazon ECS Fargate\nPython / FastAPI\nRAGオーケストレーション・チャット管理"]
        end

        subgraph PRIV_DB["Private Subnet（Database）"]
            RDS[("🐘 Amazon RDS\nPostgreSQL 15\nチャット履歴・ユーザー情報")]
        end
    end

    %% ── AI 基盤 ──
    subgraph AI["🤖 AI・ナレッジ基盤（AWS マネージド）"]
        BEDROCK["🧠 Amazon Bedrock\nClaude 3 Haiku / Sonnet\nLLM推論"]
        KB["📚 Bedrock Knowledge Bases\nRAG・ベクトル検索管理"]
        OSS["🔍 OpenSearch Serverless\nベクトルDB（Bedrock管理）"]
    end

    %% ── ストレージ ──
    subgraph STORAGE["📦 Amazon S3"]
        S3_FRONT["🪣 S3 Bucket\nReact 静的ファイル"]
        S3_KB["🪣 S3 Bucket\nナレッジベース\nMDファイル群"]
    end

    %% ── 将来連携 ──
    SF(["☁️ Salesforce\nワークフロー\n（Phase 2）"])

    %% ── フロー ──
    User -->|HTTPS| CF
    CF -->|静的ファイル配信| S3_FRONT
    CF -->|APIリクエスト| WAF
    WAF --> APIGW
    APIGW -->|ルーティング| ECS

    ECS -->|チャット履歴 読み書き| RDS
    ECS -->|RAGクエリ| KB
    ECS -->|LLM推論リクエスト| BEDROCK

    KB -->|ベクトル検索| OSS
    KB -->|ドキュメント取得| S3_KB
    BEDROCK <-->|コンテキスト付き推論| KB

    ECS -.->|申請データ連携\n（Phase 2）| SF

    %% ── スタイル ──
    classDef orange fill:#FF9900,stroke:#232F3E,color:#fff,font-weight:bold
    classDef green  fill:#3F8624,stroke:#232F3E,color:#fff,font-weight:bold
    classDef purple fill:#8C4FFF,stroke:#232F3E,color:#fff,font-weight:bold
    classDef red    fill:#DD344C,stroke:#232F3E,color:#fff,font-weight:bold
    classDef blue   fill:#1A73E8,stroke:#0D47A1,color:#fff,font-weight:bold
    classDef gray   fill:#888,stroke:#555,color:#fff

    class CF,APIGW,ECS,S3_FRONT,S3_KB orange
    class RDS green
    class BEDROCK,KB,OSS purple
    class WAF red
    class User blue
    class SF gray
```

---

## 2. リクエスト・データフロー（シーケンス図）

```mermaid
sequenceDiagram
    actor User as 👤 一般社員
    participant CF   as CloudFront
    participant APIGW as API Gateway
    participant ECS  as ECS (FastAPI)
    participant RDS  as RDS PostgreSQL
    participant KB   as Bedrock Knowledge Bases
    participant LLM  as Bedrock (Claude)

    User->>CF: 質問入力「PCの申請方法は？」
    CF->>APIGW: HTTPS フォワード
    APIGW->>ECS: POST /api/v1/chat/sessions/{id}/messages

    ECS->>RDS: 会話履歴取得（直近Nターン）
    RDS-->>ECS: 履歴データ返却

    ECS->>KB: RAGクエリ（質問のベクトル検索）
    KB-->>ECS: 関連MDファイルチャンク返却

    ECS->>LLM: プロンプト送信<br/>（システムプロンプト＋履歴＋RAGコンテキスト＋質問）
    LLM-->>ECS: 回答ストリーミング

    ECS-->>User: Server-Sent Events でリアルタイム配信
    ECS->>RDS: 会話履歴保存（質問・回答・引用元）
```

---

## 3. ネットワーク構成

```mermaid
graph TB
    subgraph Internet["🌐 インターネット"]
        User["👤 ユーザー"]
    end

    subgraph AWS["AWS"]
        CF["CloudFront\n（エッジロケーション）"]
        WAF["WAF"]
        IGW["Internet Gateway"]

        subgraph VPC["VPC  10.0.0.0/16"]
            subgraph PUB["Public Subnet 10.0.1.0/24"]
                APIGW["API Gateway\n（VPCリンク）"]
                NAT["NAT Gateway"]
            end

            subgraph PRIV1["Private Subnet A 10.0.2.0/24"]
                ECS1["ECS Task #1"]
            end

            subgraph PRIV2["Private Subnet B 10.0.3.0/24\n（将来 Multi-AZ 用）"]
                ECS2["ECS Task #2\n（POCは任意）"]
            end

            subgraph PRIV_DB["Private Subnet DB 10.0.4.0/24"]
                RDS[("RDS PostgreSQL")]
            end
        end

        subgraph MANAGED["AWS マネージドサービス"]
            BEDROCK["Bedrock\n（LLM・Knowledge Bases）"]
            S3["S3\n（静的ファイル・MDファイル）"]
        end
    end

    User --> CF --> WAF --> IGW --> APIGW
    APIGW --> ECS1
    APIGW --> ECS2
    ECS1 & ECS2 --> RDS
    ECS1 & ECS2 --> NAT
    NAT --> BEDROCK
    NAT --> S3
```

---

## 4. セキュリティレイヤー

```mermaid
graph LR
    L1["🛡️ WAF\n不正リクエスト遮断\nSQLi・XSS防御"]
    L2["🔀 API Gateway\nJWT認証\nレート制限"]
    L3["🔑 IAM Role\nECS→Bedrock/S3\n最小権限"]
    L4["🔒 Security Group\nECS→RDS のみ許可\nインバウンド制限"]
    L5["🔐 暗号化\nRDS・S3 SSE\n通信 TLS"]

    L1 --> L2 --> L3 --> L4 --> L5
```

---

## 5. コンポーネント一覧

| サービス | 役割 | POC設定 |
|---|---|---|
| **Amazon CloudFront** | CDN・HTTPS終端・キャッシュ | 1ディストリビューション |
| **AWS WAF** | SQLi・XSS・不正アクセス防御 | CloudFrontに紐付け |
| **Amazon API Gateway** | HTTPルーティング・JWT認証・レート制限 | HTTP API |
| **Amazon ECS Fargate** | Python/FastAPIコンテナ実行 | 1タスク（0.5vCPU / 1GB） |
| **Amazon RDS PostgreSQL** | チャット履歴・ユーザー情報 | db.t3.micro / Single-AZ |
| **Amazon S3**（静的） | React ビルド成果物のホスティング | 1バケット |
| **Amazon S3**（KB） | MDファイルのナレッジソース | 1バケット |
| **Amazon Bedrock Knowledge Bases** | RAGパイプライン・ベクトル検索管理 | Titan Embeddings V2 |
| **Amazon OpenSearch Serverless** | ベクトルDB（Bedrock自動管理） | Bedrock KB に紐付け |
| **Amazon Bedrock（Claude）** | LLM推論エンジン | Claude 3 Haiku（POCコスト優先）|

---

## 6. POC → 本番 移行ポイント

| 項目 | POC | 本番 |
|---|---|---|
| ECS タスク数 | 1 | Auto Scaling（2〜10） |
| RDS | Single-AZ / db.t3.micro | Multi-AZ / db.t3.medium〜 |
| 認証 | 簡易JWT | Amazon Cognito / 社内IdP連携 |
| Bedrock モデル | Claude 3 Haiku | Claude 3 Sonnet（精度優先） |
| 監視 | CloudWatch 基本メトリクス | CloudWatch Alarms + ダッシュボード |

---

*アイコン凡例: 🧠 AI/ML | 🐳 コンテナ | 🐘 DB | 🪣 ストレージ | 🔀 ネットワーク | 🛡️ セキュリティ*
