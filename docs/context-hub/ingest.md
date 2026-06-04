# データ取り込み

!!! abstract "要約"
    Context-Hub は **Slack / Backlog / Redmine / Gmail** の外部ソースと、**Inbox フォルダ**（`.md` / `.txt` をドロップ）から文脈を取り込みます。手動実行は `context-hub ingest <source>`、全有効ソースの一括同期は **`context-hub ingest all`**。`serve` を起動しておけば**有効ソースを自動で定期同期**します（`CH_SOURCE_SYNC_ENABLED`、各ソースの `syncInterval`、最小 5 分）。議事録は取り込み時に **on-prem LLM でタスクを自動抽出**し、生トランスクリプトを外部に出しません。

---

## 取り込み経路の一覧

| 経路 | 入力 | 取り込み方法 | モード |
|---|---|---|---|
| **Slack** | メッセージ・スレッド | `ingest slack` / `POST /api/v1/sources/slack/sync` | mock / live |
| **Backlog** | 課題・Wiki | `ingest backlog` / `POST /api/v1/sources/backlog/sync` | mock / live |
| **Redmine** | 課題・Wiki | `ingest redmine` / `POST /api/v1/sources/redmine/sync` | mock / live |
| **Gmail** | ラベル付きメール | `ingest gmail` / `POST /api/v1/sources/gmail/sync` | mock / live |
| **Inbox フォルダ** | `.md` / `.txt` ファイル | フォルダにドロップ（監視）/ `ingest inbox` | （モード無関係） |
| **手動 REST** | 任意テキスト・ファイル | `POST /api/v1/documents` / `/documents/upload` | — |

!!! note "mock と live"
    - **mock**: バンドルされたフィクスチャを使い、API キー不要で動作確認できます（`quickstart` / `personal` の既定）。
    - **live**: `.env` の認証情報で実 API を叩きます（`production` の既定、または `--mode live` / `INGEST_MODE=live`）。

## CLI で取り込む

### 単一ソース

```bash
# Slack を mock で取り込み
context-hub ingest slack

# Backlog を live（実 API）で取り込み
context-hub ingest backlog --mode live

# プロジェクトが複数あるときは対象を指定
context-hub ingest redmine --project-id <uuid>
```

- 引数: `slack | backlog | redmine | gmail | inbox | all`
- `--mode` / `-m`: `mock`（既定）/ `live`（`inbox` では無視）
- `--project-id`: 対象プロジェクト UUID（複数存在する場合に必須）

!!! warning "CLI 取り込みは SQLite プロファイル向け"
    CLI の `ingest` / `query` は SQLite プロファイル（`quickstart` / `personal`）で動作します。PostgreSQL（`production`）では HTTP API の `POST /api/v1/sources/*/sync` を使ってください。

### 全ソース一括同期 — `ingest all`

`ingest all` は、そのプロジェクトの**有効な外部ソースをすべて 1 回で同期**し、`CH_INBOX_DIR` が設定されていれば Inbox も走査します。1 つのソースが失敗してもログに残して残りは継続するため、1 つの認証ミスが全体をブロックしません。**スケジュール実行に置くべきコマンド**です。

```bash
CH_PROFILE=personal INGEST_MODE=live context-hub ingest all
# Ingest-all: project_id='…', 2 enabled source(s), mode=live.
#   + slack: status=completed items=12
#   + backlog: status=completed items=3
# Ingest-all complete. project_id='…' succeeded=2 failed=0
```

## serve 常駐の自動同期

外部スケジューラを用意しなくても、`context-hub serve` を起動しておけば自動同期が回ります。起動時に**全プロジェクトの有効な外部ソースごとにインターバルジョブを登録**し、各ソースの `syncInterval`（最小 5 分、未設定時は既定 15 分）で再同期します。Inbox フォルダ監視も同じ仕組みで動きます。ジョブの失敗は隔離され、スケジューラ全体は落ちません。

```bash
# 自動同期を無効化（.env または環境変数）
CH_SOURCE_SYNC_ENABLED=false
```

!!! tip "自動化はどちらか一方に"
    自動化方式は **`serve` の内蔵スケジューラ**（常時起動なら最も簡単）か、**外部の `ingest all` ジョブ**（再起動をまたいで堅牢／`serve` 常駐不要）の **どちらか一方**を選んでください。両方有効にすると二重に同期されます。macOS の launchd / cron / systemd / Windows Task Scheduler のレシピは配布物の `examples/launchd/` にあります。

## Inbox フォルダ（ドロップするだけ）

`CH_INBOX_DIR` を設定し、対応するサブフォルダにファイルを置くだけで、バックグラウンドジョブが `CH_INBOX_POLL_SECONDS`（既定 60 秒）ごとに走査して Document として upsert します。**実行するコマンドも叩く API もありません。**

```bash
# .env
CH_INBOX_DIR=~/.context-hub/inbox
CH_INBOX_POLL_SECONDS=60
# CH_PROJECT_ID=...   # プロジェクトが複数ある場合のみ必要
```

!!! note "Windows のパス"
    Windows では `~/...` の代わりに `C:\Users\<ユーザー>\.context-hub\inbox` のような絶対パス（または `%USERPROFILE%\.context-hub\inbox`）を指定してください。以降のパス例も同様に読み替えます。

```text
~/.context-hub/inbox/
  meeting/
    2026-05-20-weekly.md
    2026/05/20-detail.md      # ネストしたサブフォルダも可
  file/
    product-spec.md
  email/
    saved-thread.txt
```

ルール:

- **受理する拡張子は `.md` と `.txt` のみ。** PDF / PowerPoint / docx は事前に Markdown へ変換してください（ファイルアップロードは別経路）。
- 先頭の Markdown `# H1` をタイトルに使用。なければファイル名（拡張子なし）。
- `external_id = "<source_type>/<相対パス>"`。同じファイルを編集すると既存ドキュメントを upsert。変更がなければスキップ（再埋め込みコストなし）。
- 隠しファイル（`.DS_Store` など）や未知のサブフォルダは無視。
- `CH_INBOX_DIR` 未設定なら監視は無効。

一回だけ走査したいときは CLI で実行できます（監視と同じコードパス）。

```bash
context-hub ingest inbox
```

## Gmail 取り込み（ラベルベースのオプトイン）

既定クエリ `label:context-hub` により、**自分で `context-hub` ラベルを付けたメールだけ**が取り込まれます。私的なメールはストアに入りません。任意の Gmail 検索構文で範囲を変更できます。

```bash
# 1. 依存を追加
pip install 'yohakuforce-context-hub[gmail]'

# 2. Google Cloud Console で OAuth 2.0 Desktop クライアントを作成し credentials.json を取得
#    （Gmail API を有効化）。例: ~/.context-hub/gmail/credentials.json

# 3. .env に設定
GMAIL_CREDENTIALS_FILE=~/.context-hub/gmail/credentials.json
GMAIL_TOKEN_FILE=~/.context-hub/gmail/token.json
GMAIL_QUERY=label:context-hub
INGEST_MODE=live

# 4. 初回はブラウザで同意。以降は refresh token が token.json にキャッシュされます
context-hub ingest gmail --mode live
```

スケジュール同期したい場合は、Slack/Backlog/Redmine と同じ要領で `source_type=EMAIL` の `SourceConfig` をプロジェクトに追加します。

## 議事録 → タスク自動抽出（生データ非送出）

会議ドキュメントの**取り込み時**に、`LLM_PROVIDER` で選んだ **on-prem LLM**（Ollama / Claude Code 等）がアクションタスクを抽出し、ドキュメントに永続化します。**生トランスクリプトは外部サービスに送りません。**

- 抽出結果は `extractedTasks` として `title` / `suggestedAssignee` / `suggestedDueDate` を持ちます。
- `GET /api/v1/projects/{projectId}/meetings/{meetingId}` と MCP の `get_meeting` が返します。
- 取り込み時に確定・永続化されるため、**読み直しても結果は不変**（取りこぼし防止）。

詳細なレスポンス形は [REST / MCP API](api.md) を参照してください。

## 手動ドキュメント取り込み（補足）

議事録・メモ・メール本文など、ユーザー起点のテキストは REST から直接投入できます。

```bash
# テキストを直接投入（upsert は project_id + source_type + external_id）
curl -X POST http://127.0.0.1:8000/api/v1/documents \
  -H "X-Api-Key: $CONTEXT_HUB_API_KEY" -H "Content-Type: application/json" \
  -d '{"project_id":"proj-001","source_type":"meeting","title":"2026-05-20 weekly",
       "text":"Decision: ...","external_id":"meeting-2026-05-20","author":"koya"}'

# ファイルをアップロード（.md/.txt は常時、.pdf/.docx は [documents] extra が必要、最大 10 MiB）
curl -X POST http://127.0.0.1:8000/api/v1/documents/upload \
  -H "X-Api-Key: $CONTEXT_HUB_API_KEY" \
  -F "project_id=proj-001" -F "source_type=file" -F "file=@./specs/phase1.pdf"
```

`source_type` は `meeting | file | email`。Slack / Backlog / Redmine のコンテンツは専用の `/sources/*/sync` 経由で取り込みます。
