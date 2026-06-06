# TikTokライバーマネジメントシステム
# TikTok Livers Management System (Docker Orchestration)

## 概要
このプロジェクトは、TikTokライバーのマネジメント業務を効率化し、所属ライバーへの情報還元および自社ブランディングを目的としたシステムです。
以下の3つの主要なコンポーネントで構成されており、Docker Composeを用いてコンテナベースで運用されます。
本リポジトリは、TikTokライバー管理システムを構成する各サービスのオーケストレーションを管理します。

1.  **自社コーポレートサイト (`corporate-site`)**:
    *   会社概要の表示、Tikfinity連携によるポイントガチャ機能を提供します。
2.  **ライバー向けランキングサイト (`tiktok-ranking`)**:
    *   所属ライバーの活動状況をランキング形式で表示し、モチベーション向上を促します。簡易認証機能付き。
3.  **データ抽出ワーカー (`data-fetcher`)**:
    *   手動で取得した TikTok Live Backstage の活動データを読み込み、Googleスプレッドシートへの同期とランキングサイトへのデータ提供を行います。週次で自動実行されます。
## 構成ファイル
本プロジェクトは、役割に応じて Docker Compose ファイルを分割しています。
保守性と運用の柔軟性を高めるため、共通基盤となる「インフラ層」と、業務ロジックを担う「アプリケーション層」を分離したマルチ・ファイル構成を採用しています。

専用のデータベースサーバーは使用せず、Googleスプレッドシートおよび共有ボリューム上のローカルファイル（CSV/Excel）をデータストレージとして活用することで、運用保守の簡略化を図っています。
1. **compose.infra.yaml**: 共通ネットワーク、リバースプロキシ (Traefik)、コンテナ管理 (Portainer) などの基盤サービスを定義します。
2. **compose.app.yaml**: 各フロントエンド、認証基盤 (Keycloak)、データ抽出 Worker など、業務ロジックを担うアプリケーションサービスを定義します。
3. **compose.prod.yaml**: 本番環境用の SSL/TLS 設定や HTTP リダイレクト設定などを定義します（オーバーライド用）。

---

## 導入手順

### 1. リポジトリのクローンと初期化
```bash
git clone https://github.com/your-username/docker-compose.git
cd docker-compose
git submodule update --init --recursive
```

### 2. 環境変数の設定
`secrets/` ディレクトリ内に以下の 4 つの環境変数ファイルをそれぞれ作成し、設定を行います。
設定値のテンプレートは `secrets/.env.example` を参考にしてください。

1.  **`.env.common`**: 全サービス共通の設定（タイムゾーンなど）
2.  **`.env.google`**: Google Sheets API 関連の共通設定
3.  **`.env.corporate`**: コーポレートサイト専用の設定（Tikfinity トークンなど）
4.  **`.env.fetcher`**: データ抽出ワーカー専用の設定（Discord通知など）

> **注意**: `secrets/` フォルダ内のファイルは機密情報を含むため、Git の管理対象から除外（`.gitignore` への追加）を強く推奨します。

### 3. Googleサービスアカウントキーの配置
Googleスプレッドシート連携のために、Google Cloud Platformで作成したサービスアカウントのキーファイルを `secrets/service-account.json` としてプロジェクトルートに配置してください。

### 4. SSL証明書保存用ファイルの作成 (acme.json)
Traefikが取得したSSL証明書を保存するためのファイルを作成し、適切な権限を設定します。このファイルは所有者のみが読み書き可能（Linux環境では権限 `600`）である必要があります。

```bash
mkdir letsencrypt
touch letsencrypt/acme.json
chmod 600 letsencrypt/acme.json
```

> **注意**: Windows環境（Docker Desktop）を使用している場合でも、空のファイルを作成しておくことで、Docker起動時にファイルではなくディレクトリとして自動生成されてしまう問題を回避できます。

```text
docker-compose/
├── secrets/            # Git管理対象外
│   ├── .env.common     # 共通設定
│   ├── .env.google     # Google API設定
│   ├── .env.corporate  # コーポレート専用
│   ├── .env.fetcher    # ワーカー専用
│   ├── .env.example    # 設定テンプレート
│   └── service-account.json # Google認証キー
├── letsencrypt/
│   └── acme.json       # SSL証明書保存用 (要chmod 600)
├── services/
└── ...
```

## コンテナの起動手順

### A. 開発環境 (HTTP / localhost)
1. **環境変数の準備**: `cp secrets/.env.example .env`
2. **起動**: (認証を利用する場合、`.env` で `NEXT_PUBLIC_AUTH_ENABLED=true` とした上で `--profile auth` を指定します)
   ```bash
   # 認証あり
   docker compose -f compose.infra.yaml -f compose.app.yaml --profile auth up -d --build

   # 認証なし
   docker compose -f compose.infra.yaml -f compose.app.yaml up -d --build
   ```
3. **アクセス**: http://localhost, http://ranking.localhost

### B. 本番環境 (HTTPS / Domain)
1. **環境変数の設定**: `.env` の `base_domain` をドメイン名に変更し、`PROTOCOL=https` とします。
2. **起動**: (認証を利用する場合は同様に `--profile auth` を追加してください)
   ```bash
   docker compose -f compose.infra.yaml -f compose.app.yaml -f compose.prod.yaml up -d --build
   ```
3. **アクセス**: `https://your-domain.com`

---

## コンテナの管理

**ログの確認:** `docker compose logs -f`
**停止:** `docker compose down`

## 定期実行の設定 (`data-fetcher`)
`data-fetcher` コンテナは、詳細設計書に基づき「毎週月曜日 26:00 (JST)」に配置されたデータの加工・同期を実行します。
このスケジュール実行は、ホストマシンの OS に合わせて設定してください。

### 運用フロー
1. 管理者が TikTok Live Backstage からデータを手動でエクスポートし、ホストの `./data` フォルダに配置します。
2. スケジュールされたジョブ（または手動実行）により、`docker compose run --rm worker-fetcher` が実行されます。
3. コンテナがデータを加工し、Google スプレッドシートを更新、アーカイブを作成します。

※ `--rm` オプションにより、実行終了後にタスク用コンテナが自動削除され、リソースを節約します。

## 注意事項
*   **Google Sheets APIのレート制限**: Google Sheets APIには利用制限があります。過度なアクセスを避けるため、各サービスのキャッシュ戦略を適切に設定してください。
*   **利用規約の遵守**: 本システムの使用にあたっては、「TikTok Live Backstage利用規約」を遵守してください。特に、第7条（禁止事項）および第10条（秘密保持）に留意し、正当な権限に基づいた内部管理目的でのみ利用してください。自動化ツールの利用は、プラットフォームの負荷にならないよう適切に設定してください。

---

ご不明な点がありましたら、お問い合わせください。
## 各サービスへのアクセス
- コーポレートサイト: http://localhost
- ランキングサイト: http://ranking.localhost
- Keycloak 管理画面: http://auth.localhost
- Portainer (管理用): http://localhost:9000