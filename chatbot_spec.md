# ITヘルプデスク AIチャットボット 仕様書

**バージョン**: 1.0.0  
**作成日**: 2026-04-07  
**フェーズ**: POC（概念実証）

---

## 1. プロジェクト概要

### 1.1 目的

社内のITヘルプデスク業務・PCおよびスマートフォンのキッティング作業マニュアル・各種ワークフロー申請内容の作成を支援するAIチャットボットを開発する。

既存のMarkdown（.md）形式のマニュアルをナレッジベースとして活用し、対話形式で社員の質問に回答することで、ヘルプデスク担当者の工数削減と申請業務の効率化を実現する。

### 1.2 対象ユーザー

| ユーザー区分 | 説明 |
|---|---|
| 一般社員 | PCや通信機器などの利用申請を行う従業員 |
| ITヘルプデスク担当者 | ナレッジベース（MDファイル）の管理者（将来フェーズ） |

### 1.3 フェーズ計画

| フェーズ | 内容 | 時期 |
|---|---|---|
| Phase 1（本仕様） | POC：コア機能実装・AWS構築 | 現在 |
| Phase 2 | Salesforce ワークフロー連携・自動申請機能 | 将来 |

---

## 2. 機能要件

### 2.1 チャット機能

| 機能ID | 機能名 | 説明 | 優先度 |
|---|---|---|---|
| CHAT-01 | 対話形式Q&A | MDファイルを元にユーザーの質問に回答する | 必須 |
| CHAT-02 | チャット履歴保持 | セッション内・セッション跨ぎでの会話履歴を保持する | 必須 |
| CHAT-03 | ストリーミング応答 | 回答をリアルタイムにストリーミング表示する | 推奨 |
| CHAT-04 | 引用元表示 | 回答の根拠となったMDファイル名・箇所を表示する | 推奨 |

### 2.2 ユーザー管理機能

| 機能ID | 機能名 | 説明 | 優先度 |
|---|---|---|---|
| USER-01 | ユーザー認証 | 社員IDによるログイン（POCはシンプル認証） | 必須 |
| USER-02 | 履歴削除 | ユーザーが自身のチャット履歴を削除できる | 必須 |
| USER-03 | セッション管理 | セッションタイムアウト・ログアウト機能 | 必須 |

### 2.3 ファイル管理機能

| 機能ID | 機能名 | 説明 | 優先度 |
|---|---|---|---|
| FILE-01 | チャット履歴エクスポート | 会話履歴をMarkdown/PDF形式でダウンロード | 必須 |
| FILE-02 | ファイルインポート | ナレッジベース用MDファイルのアップロード（管理者向け） | 推奨 |
| FILE-03 | ナレッジベース同期 | S3アップロード後にBedrockナレッジベースを再同期する | 推奨 |

### 2.4 将来連携機能（Phase 2）

| 機能ID | 機能名 | 説明 |
|---|---|---|
| SF-01 | Salesforce 申請連携 | チャットで収集した申請情報をSalesforceワークフローに自動投入 |
| SF-02 | 申請ステータス照会 | Salesforce上の申請ステータスをチャットで確認 |

---

## 3. 非機能要件

### 3.1 パフォーマンス（POCスコープ）

| 項目 | 目標値 |
|---|---|
| チャット応答開始（TTFB） | 3秒以内 |
| 同時接続ユーザー数 | 〜10名（POC） |
| ナレッジベースファイル数 | 〜100ファイル（POC） |

### 3.2 セキュリティ

| 項目 | 方針 |
|---|---|
| 通信暗号化 | HTTPS（CloudFront + ACM） |
| 認証 | JWT トークン認証 |
| ナレッジベースアクセス | IAMロールによる最小権限制御 |
| チャットデータ | RDS暗号化・S3暗号化（SSE-S3） |

### 3.3 可用性（POCスコープ）

- ECS: タスク最小1台（本番化時にAuto Scalingを設定）
- RDS: Single-AZ（POC）→ Multi-AZ（本番）

---

## 4. 技術スタック

### 4.1 フロントエンド

| 項目 | 採用技術 |
|---|---|
| フレームワーク | React 18 |
| 言語 | TypeScript |
| UIライブラリ | Tailwind CSS / shadcn/ui |
| チャットUI | カスタムコンポーネント（ストリーミング対応） |
| ホスティング | Amazon S3 + CloudFront |

### 4.2 バックエンド（API）

| 項目 | 採用技術 |
|---|---|
| 言語 | Python 3.12 |
| フレームワーク | FastAPI |
| コンテナ | Docker / Amazon ECS（Fargate） |
| APIゲートウェイ | Amazon API Gateway（HTTP API） |
| ORM | SQLAlchemy |

### 4.3 AIエンジン

| 項目 | 採用技術 |
|---|---|
| LLM | Amazon Bedrock（Claude 3 Sonnet / Haiku） |
| RAG | Amazon Bedrock Knowledge Bases |
| ベクトルDB | Amazon OpenSearch Serverless（Bedrock管理） |
| Embeddingモデル | Amazon Titan Embeddings V2 |
| ナレッジソース | Amazon S3（MDファイル格納） |

### 4.4 データストア

| 項目 | 採用技術 | 用途 |
|---|---|---|
| RDB | Amazon RDS for PostgreSQL 15 | チャット履歴・ユーザー情報 |
| オブジェクトストレージ | Amazon S3 | MDファイル・静的ファイル |

---

## 5. データモデル

### 5.1 テーブル定義

#### users テーブル

```sql
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id VARCHAR(50) UNIQUE NOT NULL,  -- 社員ID
    display_name VARCHAR(100) NOT NULL,
    created_at  TIMESTAMP DEFAULT NOW(),
    last_login  TIMESTAMP
);
```

#### sessions テーブル

```sql
CREATE TABLE sessions (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    title       VARCHAR(200),                 -- 会話タイトル（最初の質問から自動生成）
    created_at  TIMESTAMP DEFAULT NOW(),
    updated_at  TIMESTAMP DEFAULT NOW()
);
```

#### messages テーブル

```sql
CREATE TABLE messages (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id  UUID REFERENCES sessions(id) ON DELETE CASCADE,
    role        VARCHAR(10) NOT NULL,          -- 'user' | 'assistant'
    content     TEXT NOT NULL,
    sources     JSONB,                         -- 引用元MDファイル情報
    created_at  TIMESTAMP DEFAULT NOW()
);
```

---

## 6. API仕様

### 6.1 主要エンドポイント

| メソッド | パス | 説明 |
|---|---|---|
| POST | `/api/v1/auth/login` | ログイン・JWTトークン発行 |
| POST | `/api/v1/chat/sessions` | 新規チャットセッション作成 |
| GET | `/api/v1/chat/sessions` | セッション一覧取得 |
| POST | `/api/v1/chat/sessions/{id}/messages` | メッセージ送信・AI回答取得（ストリーミング） |
| GET | `/api/v1/chat/sessions/{id}/messages` | メッセージ履歴取得 |
| DELETE | `/api/v1/chat/sessions/{id}` | セッション削除 |
| DELETE | `/api/v1/users/me/history` | ユーザー全履歴削除 |
| GET | `/api/v1/chat/sessions/{id}/export` | チャット履歴エクスポート |

### 6.2 チャットメッセージ送信（ストリーミング）

**Request**
```json
POST /api/v1/chat/sessions/{session_id}/messages
Authorization: Bearer {jwt_token}

{
  "content": "PCの利用申請はどうすれば良いですか？"
}
```

**Response（Server-Sent Events）**
```
data: {"type": "content", "delta": "PC利用申請は"}
data: {"type": "content", "delta": "以下の手順で"}
data: {"type": "sources", "sources": [{"file": "pc_request.md", "section": "## 申請手順"}]}
data: {"type": "done"}
```

---

## 7. ナレッジベース管理

### 7.1 MDファイル構成（S3）

```
s3://[bucket-name]/knowledge-base/
├── helpdesk/
│   ├── pc_request.md          # PC利用申請手順
│   ├── smartphone_request.md  # スマートフォン申請手順
│   └── network_trouble.md     # ネットワークトラブルシューティング
├── kitting/
│   ├── windows_kitting.md     # Windows端末キッティング手順
│   └── ios_kitting.md         # iOS端末キッティング手順
└── workflow/
    ├── approval_flow.md       # 申請ワークフロー説明
    └── faq.md                 # よくある質問
```

### 7.2 Bedrock Knowledge Base 設定

| 項目 | 設定値 |
|---|---|
| データソース | S3（上記パス） |
| チャンキング戦略 | 固定サイズ（512トークン、オーバーラップ20%） |
| 同期タイミング | ファイルアップロード後に手動または自動トリガー |

---

## 8. 画面構成

### 8.1 画面一覧

| 画面名 | パス | 説明 |
|---|---|---|
| ログイン画面 | `/login` | 社員ID・パスワード入力 |
| チャット画面 | `/chat` | メイン対話画面 |
| 履歴管理画面 | `/history` | セッション一覧・削除 |

### 8.2 チャット画面レイアウト

```
┌─────────────────────────────────────────┐
│  [ロゴ]  ITヘルプデスク AI  [ユーザー名] [ログアウト] │
├──────────┬──────────────────────────────┤
│ 会話履歴  │  チャットエリア               │
│          │  ┌──────────────────────┐   │
│ > 会話1  │  │ 🤖 どのようなことでお  │   │
│   会話2  │  │ 困りですか？          │   │
│   会話3  │  └──────────────────────┘   │
│          │  ┌──────────────────────┐   │
│          │  │ 👤 PCの申請方法は？   │   │
│          │  └──────────────────────┘   │
│          │                              │
│ [+ 新規] │  [入力欄         ] [送信]   │
│ [履歴削除]│  [📎エクスポート]            │
└──────────┴──────────────────────────────┘
```

---

## 9. 開発・デプロイ方針

### 9.1 開発環境

```
docker-compose.yml で以下を起動
- frontend: React（Vite dev server）
- backend:  FastAPI（uvicorn）
- db:       PostgreSQL（ローカル）
```

### 9.2 インフラ構成（IaC）

- AWS CDK（Python）でインフラをコード管理
- GitHub Actions でCI/CD（Lint → Test → Build → Deploy）

### 9.3 POCから本番への移行ポイント

| 項目 | POC | 本番 |
|---|---|---|
| ECS タスク数 | 1 | Auto Scaling（2〜10） |
| RDS | Single-AZ | Multi-AZ |
| 認証 | 簡易JWT | Amazon Cognito / 社内IdP連携 |
| 監視 | CloudWatch基本 | CloudWatch + アラーム設定 |

---

## 10. 将来拡張（Phase 2）：Salesforce連携

### 10.1 連携方式

```
チャットボット（ECS）
  → Salesforce REST API
    → ワークフロー申請レコード自動作成
```

### 10.2 想定ユースケース

1. チャットで申請内容（機器種別・理由・必要日）を収集
2. 収集情報をSalesforceの申請フォーム項目にマッピング
3. Salesforce上に申請レコードを自動作成
4. 承認フローをSalesforce標準ワークフローで処理

---

*本仕様書はPOCフェーズ向けです。本番化に際しては要件の見直しを行うこと。*
