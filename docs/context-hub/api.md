# REST / MCP API

!!! abstract "要約"
    Context-Hub は **REST API** と **MCP サーバ**の 2 つの入り口を持ち、いずれも共有の `QueryService` を薄くラップします。REST は `/api/v1` 配下にプロジェクト・検索・取り込み・設定・状態の各エンドポイントを公開し、運用用に `/health` と `/mcp/version` を持ちます。**0.3.0 の破壊的変更で、すべての REST 応答 JSON は camelCase に統一**されました（リクエストは snake_case も後方互換で受理）。MCP は stdio トランスポートで **7 種のツール**を提供し、Claude Desktop / Claude Code から直接利用できます。

---

## 認証

`/health` と `/mcp/version` を除く全エンドポイントは **`X-Api-Key` ヘッダ**を要求します。開発（`quickstart` / `personal`）では `init` が発行した `DEV_API_KEY` が ADMIN スコープとして通ります。production では発行した consumer キーを使います（詳細は [設定GUI（/admin）](admin-gui.md) と `SECURITY.md`）。

```bash
curl -H "X-Api-Key: $CONTEXT_HUB_API_KEY" http://127.0.0.1:8000/api/v1/status
```

!!! warning "0.3.0 BREAKING — 応答は camelCase"
    全 API 応答の JSON キーが **camelCase** に統一されました（例: `project_id` → `projectId`、`external_id` → `externalId`、`source_type` → `sourceType`、`document_count` → `documentCount`）。**リクエストボディは snake_case も後方互換で受理**します。snake_case 応答に依存していた外部コンシューマは修正が必要です。

## 運用エンドポイント（認証不要）

| メソッド | パス | 説明 |
|---|---|---|
| GET | `/health` | `{"status":"ok"}` のみを返す（環境のフィンガープリンティングを避けるため最小） |
| GET | `/mcp/version` | MCP プロトコルバージョン・サーバ名・サーバ版を返す |

```bash
curl http://127.0.0.1:8000/health
# {"status":"ok"}

curl http://127.0.0.1:8000/mcp/version
# {"mcp_protocol_version":"2024-11-05","server":"context-hub","server_version":"0.3.0"}
```

!!! note "開発時のみ /docs"
    `APP_ENV` が production 以外のとき、FastAPI の対話ドキュメント（`/docs`・`/redoc`）が有効です。production では無効化されます。

## REST 主要エンドポイント

ベースパスは `/api/v1`。スコープ列は必要な権限（`READ` / `WRITE` / `ADMIN`）です。

### プロジェクト / コンテキスト

| メソッド | パス | スコープ | 説明 |
|---|---|---|---|
| GET | `/projects` | READ | プロジェクト一覧 |
| GET | `/projects/{projectId}/context` | READ | プロジェクト文脈サマリ |
| GET | `/projects/{projectId}/members` | READ | メンバー一覧（課題から集約） |
| GET | `/projects/{projectId}/meetings` | READ | 会議一覧 |
| GET | `/projects/{projectId}/meetings/{meetingId}` | READ | 会議詳細（`extractedTasks` を含む） |
| GET | `/projects/{projectId}/issues` | READ | 課題一覧（Backlog / Redmine） |

#### Sources / プロジェクト管理（ADMIN GUI のバックエンド）

| メソッド | パス | スコープ | 説明 |
|---|---|---|---|
| GET | `/projects/detailed` | ADMIN | ソース設定込みの詳細一覧 |
| POST | `/projects` | ADMIN | プロジェクト作成 |
| PUT / DELETE | `/projects/{projectId}` | ADMIN | プロジェクト更新 / 削除 |
| PUT / DELETE | `/projects/{projectId}/sources/{sourceType}` | ADMIN | ソース設定の追加・更新 / 削除 |

### 検索（ハイブリッド）

`POST /api/v1/query`（READ）。FTS + ベクトルを RRF で統合します。

```bash
curl -X POST http://127.0.0.1:8000/api/v1/query \
  -H "X-Api-Key: $CONTEXT_HUB_API_KEY" -H "Content-Type: application/json" \
  -d '{"projectId":"proj-001","query":"デプロイ手順","topK":5}'
```

リクエスト: `projectId`（必須）、`query`（必須・最大 1000 文字）、`topK`（既定 5・1〜20）、`sourceTypes`（任意）、`filters`（任意）。応答は `results[]`（`documentId` / `sourceType` / `title` / `snippet` / `score` / `relevanceReason`）と `queryEmbeddingModel`。

### 取り込み / 同期

| メソッド | パス | スコープ | 説明 |
|---|---|---|---|
| POST | `/sources/slack/sync` | WRITE | Slack 増分同期（202・非同期） |
| POST | `/sources/backlog/sync` | WRITE | Backlog 同期 |
| POST | `/sources/redmine/sync` | WRITE | Redmine 同期 |
| POST | `/sources/gmail/sync` | WRITE | Gmail 同期 |
| GET | `/sources/jobs/{jobId}` | READ | 同期ジョブの状態 |
| POST | `/projects/{projectId}/ingest/slack` | WRITE | スクレイピング由来の Slack メッセージを冪等 upsert（Slack ts をキー、トークン不要） |
| POST | `/documents` | WRITE | テキスト直接投入（`meeting`/`file`/`email`） |
| POST | `/documents/upload` | WRITE | ファイルアップロード（`.md`/`.txt`、`[documents]` で `.pdf`/`.docx`、最大 10 MiB） |
| GET | `/documents/upload/supported-extensions` | READ | 受理可能な拡張子の確認 |

詳しい取り込み運用は [データ取り込み](ingest.md) を参照してください。

### 設定 / 状態（Admin GUI バックエンド）

| メソッド | パス | スコープ | 説明 |
|---|---|---|---|
| GET | `/config` | ADMIN | 全接続設定の取得（シークレットはマスク） |
| PUT | `/config` | ADMIN | 設定保存（`.env` 書き込み + ホットリロード） |
| POST | `/config/test/{source}` | ADMIN | 接続テスト（Slack `auth.test` / Redmine `users/current.json` のライブ ping） |
| GET | `/status` | READ | プロファイル・取り込みモード・スケジューラ・自動同期・FTS-only 判定・Inbox・有効ソース |

```bash
curl -H "X-Api-Key: $CONTEXT_HUB_API_KEY" http://127.0.0.1:8000/api/v1/status
```

!!! note "レスポンス・エンベロープ"
    多くのエンドポイントは共通の `ApiResponse` 形（`success` / `data` / メタ情報）で返します。

## MCP ツール（7 種）

stdio トランスポートで提供されます。`tools/list` で列挙、`tools/call` で実行します。

| ツール名 | 説明 | 必須引数 |
|---|---|---|
| `get_project_context` | プロジェクト文脈サマリを取得 | `projectId`（任意: `type` = `overview`/`full`） |
| `search_context` | ハイブリッド（ベクトル + キーワード）検索 | `projectId`, `query`（任意: `topK` 既定 5） |
| `get_issues` | Backlog / Redmine の課題一覧 | `projectId`, `source`（任意: `status` 既定 `open`） |
| `get_issue_detail` | コメント込みの課題詳細 | `projectId`, `issueId` |
| `get_meeting` | 議事録と要約・`extractedTasks` を取得 | `projectId`, `meetingId` |
| `get_members` | プロジェクトのメンバー一覧 | `projectId` |
| `trigger_sync` | データソースの増分同期を起動 | `projectId`, `source` |

ツールの出力キーも camelCase です（例: `documentId` / `extractedTasks` の `suggestedAssignee` / `suggestedDueDate`）。

!!! note "MCP の認証"
    MCP stdio トランスポートは localhost 限定運用を前提とし、stdio 側では認証を強制しません（HTTP 側は `X-Api-Key` 必須）。

## Claude Desktop への登録例

`~/Library/Application Support/Claude/claude_desktop_config.json` に追記します。

```json
{
  "mcpServers": {
    "context-hub": {
      "command": "context-hub",
      "args": ["serve", "--mcp-only"],
      "env": {
        "CONTEXT_HUB_API_KEY": "your-api-key"
      }
    }
  }
}
```

!!! tip "起動互換性チェック"
    AI エージェント（Claude Desktop / Claude Code）は接続前に `GET /mcp/version` を呼んでトランスポート互換性を確認できます。連携先エージェント本体のセットアップは [AI-Project-Manager のデプロイ](../ai-project-manager/deploy.md) を参照してください。CLI 全引数は [CLI リファレンス](cli-reference.md) にあります。
