# デプロイ（Docker / PostgreSQL）

!!! abstract "このページの要約"
    AI-Project-Manager は **PostgreSQL 専用**（SQLite 非対応）で、**Docker Desktop（Windows は WSL2）** 上の `docker compose up --build` で起動します。流れは `git clone` → `.env` 作成 → `docker compose up --build` の 3 ステップです。最大の連携ポイントは、AI-PM の **`CONTEXT_HUB_API_KEY` を Context-Hub 側の `DEV_API_KEY` と完全に一致させる** こと、そして Docker コンテナからホストの Context-Hub を見るために **`CONTEXT_HUB_BASE_URL=http://host.docker.internal:8000/api/v1`** を使うことです。起動後は `http://localhost:8001/health` で疎通を確認します。

---

## 前提

| 項目 | 内容 |
|---|---|
| OS / コンテナ | Windows の場合は **WSL2 + Docker Desktop for Windows**。macOS でも Docker Desktop で動作 |
| データベース | **PostgreSQL 16**（compose が `db` サービスとして起動。AI-PM は SQLite 非対応） |
| Context-Hub | 文脈連携する場合に別途起動（[Context-Hub のドキュメント](../context-hub/index.md)）。繋がない場合はモックで動作可 |
| その他 | Git。Windows では Docker Desktop の **File sharing** で対象ドライブを共有しておく（bind mount 用） |

!!! note "ポートの割り当て"
    - AI-PM API: `:8001`
    - AI-PM の PostgreSQL: ホスト側 `:5433`（コンテナ内 `5432`。Context-Hub の DB と衝突しないよう 5433 に割り当て）
    - Context-Hub: `:8000`

---

## 1. リポジトリを取得して `.env` を作る

```powershell
git clone https://github.com/yohakuforce/ai-project-manager.git
cd ai-project-manager
copy .env.example .env   # Mac/Linux は cp .env.example .env
```

`.env` の最低限の設定は次の通りです。

```env
# --- DB（compose の db サービスに合わせる） ---
DB_NAME=ai_project_manager
DB_USER=postgres
DB_PASSWORD=<強いパスワード>            # compose が必須要求（未設定だと起動しない）
USE_DATABASE=true                       # PostgreSQL に永続化（ゲート・日報・タスク等）

# --- LLM（既定はサブスク CLI・課金 API なし） ---
LLM_PROVIDER=claude-code                # 既定値。詳細は LLM プロバイダのページ参照

# --- Context-Hub 連携 ---
CONTEXT_HUB_BASE_URL=http://host.docker.internal:8000/api/v1   # コンテナ→ホストのCH
CONTEXT_HUB_API_KEY=<Context-Hub の DEV_API_KEY と同値>
CONTEXT_HUB_USE_MOCK=false              # 実 Context-Hub に接続（モックなら true）

# --- 通知（Slack トークン未取得時の安全策） ---
NOTIFICATION_CHANNEL=local_file
```

!!! warning "★ 連携の肝: `CONTEXT_HUB_API_KEY` は Context-Hub の `DEV_API_KEY` と一致させる"
    `CONTEXT_HUB_API_KEY` は **どこかから「貰う」値ではありません**。開発時は **自分で好きな値を決め、Context-Hub 側の `DEV_API_KEY` と AI-PM 側の `CONTEXT_HUB_API_KEY` を同じ値にする** ものです。Context-Hub は開発時（`APP_ENV=development`）、リクエストの `X-Api-Key` がこの `DEV_API_KEY` と一致すれば通します。

    - Context-Hub を `context-hub init --profile quickstart` で初期化すると、画面に表示される **「Admin GUI key（DEV_API_KEY）」** を控えて、それを AI-PM の `CONTEXT_HUB_API_KEY` に設定します。
    - 後から確認するには Context-Hub の `.env` 内 `DEV_API_KEY` を見ます（PowerShell: `Select-String DEV_API_KEY .env`）。
    - **本番**（`APP_ENV=production`）では `DEV_API_KEY` は無効です。Context-Hub 側で発行したコンシューマ API キー（ハッシュ管理）を使います。

!!! warning "★ Docker コンテナからホストの Context-Hub を見る URL"
    Context-Hub をホスト側のプロセスとして起動している場合、AI-PM コンテナからは `localhost` ではホストに届きません。**`CONTEXT_HUB_BASE_URL=http://host.docker.internal:8000/api/v1`** を使ってください（末尾の `/api/v1` まで含める）。Context-Hub も Docker で同一ネットワークに置く場合はサービス名で解決します。

---

## 2. 起動（`docker compose up`）

```bash
docker compose up --build
```

compose は次の 3 サービスを立ち上げます。

| サービス | 役割 | ポート |
|---|---|---|
| `db` | PostgreSQL 16（`pg_data` ボリュームで永続化） | ホスト `5433` → コンテナ `5432` |
| `migrate` | `alembic upgrade head` を起動時に **一度だけ** 実行（DB スキーマ作成） | — |
| `app` | FastAPI（`uvicorn ... --port 8001 --reload`） | `8001` |

`db` のヘルスチェックが通ってから `migrate` と `app` が起動する依存関係になっています。

!!! tip "Docker を使わないローカル起動"
    Docker を使わない場合は、ローカルで PostgreSQL を `5433` で立て、`DATABASE_URL` を実値にして次を実行します（SQLite 非対応のため Postgres は必須）。

    ```bash
    pip install -r requirements.txt
    alembic upgrade head
    uvicorn src.api.app:app --host 127.0.0.1 --port 8001
    ```

---

## 3. 動作確認

ブラウザまたは `curl` で次を確認します。

| URL | 内容 | 認証 |
|---|---|---|
| `http://localhost:8001/health` | ヘルスチェック（`{"status":"ok"}`） | 不要 |
| `http://localhost:8001/settings` | 設定 GUI（全項目編集） | 不要・localhost 専用 |
| `http://localhost:8001/register` | プロジェクト / メンバー登録 GUI | 不要・localhost 専用 |
| `http://localhost:8001/guide` | 運用ガイド | 不要・localhost 専用 |
| `http://localhost:8001/docs` | API ドキュメント（OpenAPI） | 不要（本番では無効化） |

```bash
curl http://localhost:8001/health
# → {"status":"ok","version":"0.1.0"}
```

!!! note "業務 API の認証"
    `/api/v1/...` の業務エンドポイントはすべて **`X-Api-Key` ヘッダー必須**です。値は設定の **`app_secret_key`**（既定 `dev-secret-change-in-production`）です。本番では `openssl rand -hex 32` でランダム値に変更してください。`/health` `/docs` `/settings` `/register` `/guide` は認証不要です。

---

## 4. 結線確認（手動スモーク）

1. Context-Hub にサンプルデータが投入済みであること。
2. AI-PM から Context-Hub の課題が取得できること（`CONTEXT_HUB_USE_MOCK=false` で 500 / 接続エラーが出ないこと）。`/settings` の「**Context-Hub 接続テスト**」ボタンでも確認できます。
3. 全 7 能力は `SCHEDULER_ENABLED=true` のスケジューラで稼働します（[7 つの能力](capabilities.md)）。

---

## 5. 本番化のチェックリスト

- `APP_ENV=production` / `LOG_LEVEL=INFO`
- `app_secret_key` と `jwt_secret` をランダム値に（`openssl rand -hex 32`）
- `USE_DATABASE=true` ＋ `DATABASE_URL`（compose の DB 設定と一致）→ `alembic upgrade head`
- `SCHEDULER_TIMEZONE=Asia/Tokyo`
- 通知チャンネルを実値に（Slack 等）、Context-Hub は本番キーに

詳しい連携手順は [設定 GUI](gui.md)、プロバイダ選定は [LLM プロバイダ](llm-providers.md) を参照してください。
