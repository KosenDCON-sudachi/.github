# 全体アーキテクチャ

```mermaid
flowchart LR
  %% ==============
  %% Device Layer
  %% ==============
  subgraph Device[デバイス層: ARグラス / RasPi]
    mic[マイク]
    cam[カメラ]
    disp[AR表示]
    pi[RasPi Python]
  end

  subgraph Net[ネットワーク]
    wifi[施設Wi-Fi/LAN]
  end

  %% ==============
  %% Supabase Core
  %% ==============
  subgraph Supabase[Supabase]
    edge[Edge Functions API/BFF]
    db[(Postgres DB)]
    st[(Storage: audio/image)]
    rt[Realtime 任意]
  end

  %% ==============
  %% Worker Layer
  %% ==============
  subgraph Worker[Worker 常駐/バッチ]
    wkr[ASR/正規化/要約/注意点生成]
    llm[LLM API]
    asr[Whisper ローカルorAPI]
  end

  %% ==============
  %% Sheets
  %% ==============
  subgraph Sheets[Google Sheets 編集UI]
    sheet[固定情報/編集要約/注意点 任意]
  end

  %% ==============
  %% Frontend
  %% ==============
  subgraph FE[フロントエンド]
    web[Flutter Web ダッシュボード]
  end

  %% Device -> Network -> API
  mic --> pi
  cam --> pi
  pi --> wifi
  wifi --> edge

  %% Upload flow audio
  edge -->|署名URL発行| pi
  pi -->|PUT| st
  pi -->|ジョブ作成| edge

  %% API -> DB
  edge --> db
  edge -->|必要ならpush| rt

  %% Worker picks jobs
  db -->|jobs pending| wkr
  st -->|音声/画像取得| wkr

  %% Worker processing
  wkr --> asr
  wkr --> llm
  wkr -->|timeline/daily_points更新| db
  wkr -->|必要なら要約を書き戻し| sheet

  %% Sheets sync
  sheet -->|定期同期 Upsert| wkr
  wkr -->|residents更新| db

  %% Frontend reads
  web <-->|参照/更新| edge
  web -->|参照| sheet
  web -->|参照| db

  %% AR display
  edge -->|resident summary / daily points| wifi
  wifi --> disp
```

```mermaid
erDiagram
  RESIDENTS ||--o{ TIMELINE_EVENTS : has
  RESIDENTS ||--o{ DAILY_POINTS : has
  AUDIO_ASSETS ||--o{ JOBS : triggers
  JOBS ||--o{ TIMELINE_EVENTS : produces
  
  STAFF {
   uuid id PK
   text name
   string role
  }

  RESIDENTS {
    uuid id PK
    text display_name
    text sheet_row_id
    jsonb profile_json
    timestamptz updated_at
    timestamptz created_at
  }

  AUDIO_ASSETS {
    uuid id PK
    text storage_path
    int duration_sec
    timestamptz captured_at
    text device_id
    timestamptz created_at
  }

  JOBS {
    uuid id PK
    text type
    text status
    jsonb payload_json
    timestamptz locked_at
    text locked_by
    int attempts
    text error
    timestamptz created_at
    timestamptz updated_at
    uuid audio_asset_id FK
  }

  TIMELINE_EVENTS {
    uuid id PK
    uuid resident_id FK
    uuid source_job_id FK
    text event_type
    timestamptz occurred_at
    text text_raw
    text text_normalized
    text summary
    int importance
    timestamptz created_at
  }

  DAILY_POINTS {
    uuid id PK
    uuid resident_id FK
    date date
    text content
    jsonb sources
    timestamptz created_at
    timestamptz updated_at
  }

```

申し送り

音声DB()→Whisper→

インカム会話ないよう

### 6-2. 技術スタック

- デバイス層
    - Raspberry Pi / AR グラス
    - OS: Ubuntu Server
    - 言語: Python
    - 機能：
        - マイク入力 (`arecord` / `pyaudio`)
        - 音声エンコード (`ffmpeg` / `pyav`)
        - カメラ入力（`libcamera` / `opencv-python`）
        - HTTP クライアント（`requests` / `httpx`）
        - WebSocket クライアント（`websockets` / `websocket-client`）
- バックエンド層
    - 言語: TypeScript
    - 実装候補: Node.js + Express / AWS Lambda + API Gateway / Supabase Edge Functions など
    - API:
        - `/audio/stream` → WebSocket → `audioSvc`
        - `/images` → POST → `imgSvc`
        - `/residents/{id}/summary` → GET → `residentSvc`
        - `/timeline` → GET/POST → `timelineSvc`
        - `/records` → GET/POST → `recordSvc`
    - スプレッドシート連携：
        - Google Sheets API もしくは GAS 経由でアクセス
        - 固定情報／要約テキスト／「今日の注意点」等を読み書き
- フロントエンド層
    - Flutter（Web）
    - Material Design 3
    - 状態管理: Riverpod など
    - 機能：
        - 利用者一覧・詳細表示
        - タイムライン閲覧
        - 「今日気をつけるべきポイント」の表示
        - 将来的に管理者用画面（スプレッドシートと DB の状態を確認）

### 6-3. データ保存戦略（重要）

- **Google スプレッドシート**
    - 固定情報（性格・傾向・なだめ方・持病 等）
    - 申し送り要約テキスト
    - 「今日気をつけるべきポイント」
    - 職員が直接編集できるようにする（PC 前提）
- **DB（Supabase / AWS RDS 等）**
    - タイムラインイベント（いつ／誰が／誰について／どのような内容）
    - バイタル履歴
    - イベント種別（申し送り・インカム・観察メモ 等）
- **ストレージ（Supabase Storage / S3）**
    - 申し送り音声の元データ（バイナリ）
    - 必要に応じて画像データ（様子の記録など）

### 6-4. 環境

- 環境種別
    - dev: 開発環境
    - stg: ステージング
    - prod: 本番環境
- 各環境ごとの詳細な構成・デプロイ手順は **infra リポジトリの README** に記載する。

### アーキテクチャAWS版

```mermaid
flowchart LR
  %% ==============
  %% Device Layer
  %% ==============
  subgraph Device[デバイス層: ARグラス / RasPi]
    mic[マイク]
    cam[カメラ]
    disp[AR表示]
    pi[RasPi Python]
  end

  subgraph Net[ネットワーク]
    wifi[施設Wi-Fi/LAN]
  end

  %% ==============
  %% AWS Ingress
  %% ==============
  subgraph AWS[ AWS ]
    apigw[API Gateway REST/WebSocket]
    cognito[Cognito 認証 任意]
    s3[(S3: audio/image)]
    sqs[(SQS: jobs)]
    step[Step Functions or EventBridge]
    lambda[Lambda:  API/BFF]
    worker[ECS Fargate Worker<br/>ASR/正規化/要約]
    db[(Aurora PostgreSQL or DynamoDB)]
    cw[CloudWatch Logs/Metrics]
    kms[KMS 暗号化]
  end

  %% ==============
  %% Sheets
  %% ==============
  subgraph Sheets[Google Sheets 編集UI]
    sheet[固定情報/編集要約/注意点]
  end

  %% ==============
  %% Frontend
  %% ==============
  subgraph FE[フロントエンド]
    web[Flutter Web ダッシュボード]
  end

  %% Device -> AWS
  mic --> pi
  cam --> pi
  pi --> wifi
  wifi --> apigw

  %% Auth optional
  web --> cognito
  cognito --> apigw

  %% Upload flow
  apigw -->|署名付きURL発行| lambda
  lambda --> s3
  pi -->|PUT audio/image| s3
  lambda -->|job enqueue| sqs

  %% Orchestration
  sqs --> worker
  worker -->|timeline/daily_points更新| db
  worker -->|必要なら要約を書き戻し| sheet

  %% Sheets sync
  sheet -->|定期同期 Upsert| worker
  worker -->|residents更新| db

  %% Frontend reads/writes
  web <-->|参照/更新| apigw
  apigw --> db

  %% AR display
  apigw -->|resident summary / daily points| wifi
  wifi --> disp

  %% Observability / Security
  lambda --> cw
  worker --> cw
  s3 --> kms
  db --> kms
```
