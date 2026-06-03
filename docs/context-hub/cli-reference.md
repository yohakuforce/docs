# CLI リファレンス

!!! abstract "要約"
    `context-hub` CLI は **`init` / `serve` / `ingest` / `query` / `migrate`** の 5 コマンドを提供します。標準的な流れは `init`（`.env` 生成 + `DEV_API_KEY` 発行）→ `migrate`（スキーマ適用）→ `serve`（起動）。取り込みは `ingest`、検索は `query` です。引数なしで実行するとヘルプが表示されます。

---

## 全体像

```text
context-hub init     --profile [quickstart|personal|production] [--force]
context-hub serve    [--host 127.0.0.1] [--port 8000] [--mcp-only] [--reload]
context-hub ingest   [slack|backlog|redmine|gmail|inbox|all] [--mode mock|live] [--project-id <uuid>]
context-hub query    "<text>" [--top-k 5] [--json] [--project-id <uuid>]
context-hub migrate  [--target head] [--dry-run] [--yes]
```

---

## `init`

プロファイル用の `.env` をカレントディレクトリに生成し、`data/` ディレクトリを作成します。`.env` のパーミッションは `600` に設定されます。

**用途**: プロジェクトの初期化。`quickstart` / `personal` では Admin GUI / REST 用の `DEV_API_KEY` を自動発行して出力に表示します。

| 引数 | 既定 | 説明 |
|---|---|---|
| `--profile` / `-p` | `quickstart` | `quickstart` / `personal` / `production` |
| `--force` / `-f` | false | 既存の `.env` を上書き |

```bash
context-hub init --profile quickstart
context-hub init -p production --force
```

!!! note "production は DEV_API_KEY を発行しない"
    `production`（`APP_ENV=production`）では `DEV_API_KEY` は発行されず無視されます。Admin GUI には ADMIN スコープの consumer キーを別途発行してください（`SECURITY.md`）。

## `serve`

Context-Hub サーバを起動します。**カレントディレクトリの `.env` を明示的に読み込む**ため、必ずこのコマンドで起動してください。既定動作は HTTP REST API（Admin GUI もこの上で動作）です。

**用途**: サーバ起動。`--mcp-only` で stdio MCP サーバとして起動します。

| 引数 | 既定 | 説明 |
|---|---|---|
| `--host` | `127.0.0.1` | バインドアドレス |
| `--port` | `8000` | TCP ポート |
| `--mcp-only` | false | HTTP を立てず stdio MCP サーバのみ起動 |
| `--reload` | false | uvicorn ホットリロード（開発用。production では警告） |

```bash
context-hub serve                       # HTTP REST API
context-hub serve --mcp-only            # stdio MCP サーバ（Claude Desktop 用）
context-hub serve --host 0.0.0.0 --port 9000
```

!!! warning "`--http-only` は存在しない"
    既定が HTTP REST API です。MCP サーバにしたいときだけ `--mcp-only` を付けます。

## `ingest`

指定ソースの 1 回限りの取り込みを実行します。

**用途**: Slack / Backlog / Redmine / Gmail / Inbox の手動取り込み、または `all` で全有効ソースの一括同期。

| 引数 | 既定 | 説明 |
|---|---|---|
| `source`（位置引数） | — | `slack` / `backlog` / `redmine` / `gmail` / `inbox` / `all` |
| `--mode` / `-m` | `mock` | `mock`（フィクスチャ）/ `live`（実 API）。`inbox` では無視 |
| `--project-id` | なし | 対象プロジェクト UUID（複数存在時に必須） |

各ソースの意味:

- `slack` / `backlog` / `redmine`: 各サービスから取り込み。
- `gmail`: ラベル付きメールを取り込み（live は `[gmail]` extra 必須）。
- `inbox`: `CH_INBOX_DIR` を 1 回走査し、新規・編集された `.md` / `.txt` を upsert（`--mode` は無視）。
- `all`: 有効な全ソースを 1 回で同期し、`CH_INBOX_DIR` 設定時は Inbox も走査。1 つの失敗はログに残して継続。スケジュール実行向け。

```bash
context-hub ingest slack
context-hub ingest backlog --mode live
context-hub ingest all --project-id 1f4e...   # 全ソース一括
```

!!! warning "SQLite プロファイル向け"
    CLI の `ingest` は SQLite プロファイル（`quickstart` / `personal`）で動作します。PostgreSQL（`production`）では HTTP API（`POST /api/v1/sources/*/sync`）を使ってください。

## `query`

ハイブリッド検索を実行し、結果を表示します。クエリを埋め込み、ベクトル + 全文検索でプロジェクトのドキュメントを検索します。

**用途**: コマンドラインからの動作確認・検索。

| 引数 | 既定 | 説明 |
|---|---|---|
| `text`（位置引数） | — | 検索クエリ文字列 |
| `--project-id` | なし | 検索対象プロジェクト UUID |
| `--top-k` / `-k` | 5 | 返す最大件数 |
| `--json` | false | JSON で出力 |

```bash
context-hub query "deployment checklist" --project-id <uuid> --top-k 10
context-hub query "リリース判断" --json
```

!!! note
    `ingest` と同様、`query` も SQLite プロファイル向けです。PostgreSQL では HTTP API（`POST /api/v1/query`、[REST / MCP API](api.md) 参照）を使ってください。

## `migrate`

保留中の DB マイグレーションを適用します。`DATABASE_URL` を見て、SQLite なら専用ランナー、PostgreSQL なら Alembic を自動選択します（PostgreSQL には `alembic.ini` が必要）。

**用途**: スキーマの初期化・更新。

| 引数 | 既定 | 説明 |
|---|---|---|
| `--target` | `head` | 適用先リビジョン（`head` で全保留分） |
| `--dry-run` | false | 適用せず内容のみ表示（DB パスワードはマスク） |
| `--yes` / `-y` | false | 確認プロンプトをスキップ（CI / 非対話環境用） |

```bash
context-hub migrate
context-hub migrate --target 001
context-hub migrate --dry-run
context-hub migrate --yes        # 非対話（CI）
```

!!! warning "production の安全ゲート"
    `APP_ENV=production` では適用前に確認プロンプトが表示されます。`--yes` / `-y` を渡すとスキップできます。

---

!!! tip "関連ページ"
    導入手順は [インストール](install.md)、構成の選び方は [プロファイル](profiles.md)、取り込み運用は [データ取り込み](ingest.md)、API は [REST / MCP API](api.md)、画面操作は [設定GUI（/admin）](admin-gui.md) を参照してください。
