# TikTokライバーマネジメントシステム

## 概要
このプロジェクトは、TikTokライバーのマネジメント業務を効率化し、所属ライバーへの情報還元および自社ブランディングを目的としたシステムです。
以下の3つの主要なコンポーネントで構成されており、Docker Composeを用いてコンテナベースで運用されます。

1.  **自社コーポレートサイト (`corporate-site`)**:
    *   会社概要の表示、Tikfinity連携によるポイントガチャ機能を提供します。
2.  **ライバー向けランキングサイト (`tiktok-ranking`)**:
    *   所属ライバーの活動状況をランキング形式で表示し、モチベーション向上を促します。簡易認証機能付き。
3.  **データ抽出ワーカー (`data-fetcher`)**:
    *   TikTok Live Backstageからライバーの活動データを自動抽出し、Googleスプレッドシートへの同期とランキングサイトへのデータ提供を行います。週次で自動実行されます。

専用のデータベースサーバーは使用せず、Googleスプレッドシートおよび共有ボリューム上のローカルファイル（CSV/Excel）をデータストレージとして活用することで、運用保守の簡略化を図っています。

## 必要なソフトウェアとインストール手順
本システムをローカル環境で起動・実行するためには、以下のソフトウェアが必要です。

1.  **Git**:
    *   ソースコード管理システムです。
    *   [Git公式サイト](https://git-scm.com/downloads) からお使いのOSに応じたインストーラーをダウンロードし、インストールしてください。
2.  **Docker Desktop**:
    *   コンテナ型の仮想環境を構築するためのソフトウェアです。Docker Engine、Docker Composeを含みます。
    *   [Docker公式サイト](https://www.docker.com/products/docker-desktop/) からお使いのOSに応じたインストーラーをダウンロードし、インストールしてください。
    *   インストール後、Docker Desktopアプリケーションを起動し、Dockerデーモンが動作していることを確認してください。

## プロジェクトのセットアップ

### 1. リポジトリのクローンとサブモジュールの初期化
まず、このメインリポジトリをクローンし、続いて各サービス（サブモジュール）を初期化・更新します。

```bash
git clone https://github.com/your-username/docker-compose.git # あなたのリポジトリURLに置き換えてください
cd docker-compose
git submodule update --init --recursive
```

### 2. 環境変数の設定
プロジェクトルートに `.env` ファイルを作成し、必要な環境変数を設定します。
`.env.example` を参考にしてください。（初回起動時に必要なもののみ抜粋）

```dotenv
# .env
# 共通
TIMEZONE=Asia/Tokyo

# TikTok Credentials (data-fetcherで利用)
TIKTOK_USERNAME=your_tiktok_id
TIKTOK_PASSWORD=your_tiktok_password

# Google Sheets API (corporate-site, tiktok-ranking, data-fetcherで利用)
GOOGLE_SHEET_ID=your_google_sheet_id # 各種データが保存されるメインのスプレッドシートID
# GOOGLE_MASTER_SHEET_ID=yyy # 必要であれば追加

# サービスアカウントのJSONファイルパス (コンテナ内でのパス)
# ホスト側には `secrets/service-account.json` として配置することを想定
GOOGLE_SERVICE_ACCOUNT_JSON=/app/secrets/service-account.json

# Notifications (data-fetcherで利用)
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/... # 処理通知用のDiscord Webhook URL

# Tikfinity API (corporate-siteで利用)
TIKFINITY_API_TOKEN=your_tikfinity_api_token
```

### 3. Googleサービスアカウントキーの配置
Googleスプレッドシート連携のために、Google Cloud Platformで作成したサービスアカウントのキーファイルを `secrets/service-account.json` としてプロジェクトルートに配置してください。

```text
docker-compose/
├── secrets/            # Git管理対象外
│   ├── .env.common     # 共通設定
│   ├── .env.google     # Google API設定
│   ├── .env.corporate  # コーポレート専用
│   ├── .env.fetcher    # ワーカー専用
│   ├── .env.example    # 設定テンプレート
│   └── service-account.json # Google認証キー
├── services/
└── ...
```

## コンテナの起動
プロジェクトルートディレクトリで以下のコマンドを実行します。

```bash
docker compose build # 初回起動時やDockerfile変更時に実行
docker compose up -d # バックグラウンドでコンテナを起動
```

コンテナのログを確認するには、以下のコマンドを使用します。

```bash
docker compose logs -f
```

## アクセス
システムが起動したら、以下のURLにアクセスして各サービスを確認できます。

*   **自社コーポレートサイト**: `http://localhost/`
*   **ライバー向けランキングサイト**: `http://localhost/ranking`

## 定期実行の設定 (`data-fetcher`)
`data-fetcher` コンテナは、詳細設計書に基づき「毎週月曜日 26:00 (JST)」にTikTok Live Backstageからのデータ抽出を実行します。
このスケジュール実行は、ホストマシンの OS に合わせて設定してください。

### Windows の場合 (タスクスケジューラ)
1. **タスクスケジューラ** を開き、「基本タスクの作成」を選択します。
2. **トリガー**: 「毎週」を選択し、火曜日の 02:00 AM（月曜の 26:00）に設定します。
3. **操作**: 「プログラムの開始」を選択します。
4. **プログラム/スクリプト**: `docker` (または `docker.exe` のフルパス)
5. **引数の追加**: `compose run --rm worker-fetcher`
6. **開始 (オプション)**: プロジェクトのルートディレクトリのフルパス (`e:\Users\purin\Documents\GitHub\docker-compose`)

### Linux の場合 (cron)
1. `crontab -e` を実行し、以下の行を追加します。
   ```bash
   0 2 * * 2 cd e:/Users/purin/Documents/GitHub/docker-compose && /usr/bin/docker compose run --rm worker-fetcher
   ```

※ `--rm` オプションにより、実行終了後にタスク用コンテナが自動削除され、リソースを節約します。

## 注意事項
*   **Seleniumログインプロファイル**: `data-fetcher` の初回実行時やセッション切れの際は、手動でのログインが必要になる場合があります。ログイン後に `data-fetcher/profile` ディレクトリ内にセッション情報が保存され、次回以降は自動ログインを試みます。
*   **TikTok Live BackstageのUI変更**: TikTok Live BackstageのUI変更は、`data-fetcher` のスクレイピングロジックに影響を与える可能性があります。UI変更があった場合は、`data-fetcher` リポジトリのコード修正が必要になります。
*   **Google Sheets APIのレート制限**: Google Sheets APIには利用制限があります。過度なアクセスを避けるため、各サービスのキャッシュ戦略を適切に設定してください。

---

ご不明な点がありましたら、お問い合わせください。