# 介護支援 ARシステム (Org Overview)

## この Organization について

特別養護老人ホーム向けに、ARグラス＋クラウド＋Webダッシュボードで
スタッフ同士の情報共有と記録業務を支援するシステムを開発しています。

- 背景・目的・要件の詳細: [`/docs/requirements/要件定義-v3.md`](./docs/requirements/要件定義-v3.md)
- 全体アーキテクチャ: [`/docs/architecture.md`](./docs/architecture.md)

## システム構成

- デバイス層: ARグラス (Raspberry Pi / SoC, Python)
- ネットワーク層: 施設内 Wi-Fi / LAN(仮)
- バックエンド層: API Gateway / 各種サービス (音声, 画像, タイムライン, 利用者情報, 記録)
- フロントエンド層: Dart 製管理 Web ダッシュボード(2次審査まで)

## リポジトリ一覧

| Repo | 説明 |
|------|------|
| `device-client` | ARグラス側のクライアント（音声・画像取得→API送信） |
| `backend-api` | API Gateway 配下の BFF / 認証・ルーティング |
| `svc-asr` | 音声受付〜ASR(Whisper) |
| `svc-awa` | 阿波弁→標準語 正規化AI |
| `svc-nlp` | NLP / 構造化AI |
| `svc-resident` | 利用者情報サービス |
| `svc-timeline` | タイムラインサービス |
| `svc-record` | 記録サービス |
| `frontend-web` | Dart 製 管理Webダッシュボード |
| `infra` | IaC, CI/CD 定義 |

詳細は各リポジトリの README を参照してください。

## 技術スタック

- フロントエンド: Dart / Flutter Web
- デバイス: Raspberry Pi + Ubuntu Server + Python
- バックエンド: AWS (Lambda, API Gateway, RDB, オブジェクトストレージ), 言語は TS/Go/Python
- AI: Whisper + 阿波弁正規化モデル

## 開発ルール

ここら辺はAIのやつ適当に書いてる
- ブランチ戦略: `main` / `develop` / `feature/*`
- Issue: GitHub Issues を利用。テンプレートに従って作成
- PR: 小さく・自動テストが通る状態で出す
- コードスタイル:
  - Dart: `dart format`
  - Python: `black` + `ruff`
  - TypeScript: `eslint` + `prettier`

詳細は `/docs/contributing.md` を参照。

## 環境

- `dev`: 開発環境
- `stg`: ステージング
- `prod`: 本番環境

環境ごとの詳細構成・デプロイ手順は `infra` リポジトリの README を参照してください。
