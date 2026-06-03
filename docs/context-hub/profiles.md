# プロファイル

!!! abstract "要約"
    Context-Hub には用途別の 3 プロファイルがあります。**`quickstart`**（SQLite + mock 埋め込み・ゼロ依存）、**`personal`**（SQLite + BGE-M3・単一ユーザー永続）、**`production`**（PostgreSQL + BGE-M3・本番）。プロファイルは `CH_PROFILE` 環境変数で選び、各フィールドは対応する環境変数で個別に上書きできます。`quickstart` で試し、永続運用は `personal`、チーム/本番は `production` が基本線です。

---

## 比較表

| 項目 | `quickstart` | `personal` | `production` |
|---|---|---|---|
| **用途** | ゼロ依存のローカル試用 | 単一ユーザーの永続運用 | 本番デプロイ |
| **DB** | SQLite（ファイル） | SQLite（ファイル） | PostgreSQL |
| **埋め込み** | mock（ハッシュベース） | BGE-M3* | BGE-M3* |
| **スケジューラ** | in-memory | SQLite | PostgreSQL |
| **取り込みモード（既定）** | mock | mock | live |
| **LLM プロバイダ（既定）** | ollama | ollama | claude-code |
| **`APP_ENV`** | development | development | production |
| **ログレベル** | INFO | INFO | WARNING |
| **`DEV_API_KEY`** | 自動発行・有効 | 自動発行・有効 | 無効（consumer キー必須） |
| **追加 extra** | 不要 | `[embedding]` | `[embedding]` `[postgres]` |

\* BGE-M3 は `pip install 'yohakuforce-context-hub[embedding]'` が必要です（初回に約 2.3 GB のモデルをダウンロード）。

!!! note "mock 埋め込みについて"
    `quickstart` の既定埋め込みはハッシュベースの 1024 次元ベクトルで、**意味的な精度はありません**（インストールコストゼロで配線確認するためのもの）。意味検索を効かせたい場合は `personal` / `production` で BGE-M3 を使ってください。

## プロファイル別の既定値（詳細）

### quickstart（ゼロ依存）

- `DATABASE_URL`: `sqlite+aiosqlite:///./data/context_hub.db`
- スケジューラ: `memory`（プロセス内）
- 取り込み: `mock`（フィクスチャ。API キー不要）
- 埋め込み: `mock`
- Docker も Postgres も API キーも不要。まず触ってみる段階に最適です。

### personal（単一ユーザー永続）

- `DATABASE_URL`: SQLite ファイル（quickstart と同じパス）
- スケジューラ: `sqlite`（WAL モードで永続化）
- 埋め込み: `bge-m3`（`[embedding]` extra 必須）
- 取り込み既定は `mock`。実データを引くときは `INGEST_MODE=live` か `--mode live` を指定します。

### production（本番）

- `DATABASE_URL`: `postgresql+asyncpg://...`（**既定のプレースホルダは本番で拒否**されます）
- スケジューラ: `postgres`
- 埋め込み: `bge-m3`
- 取り込み既定は `live`。
- `SECRET_KEY` は強力なランダム値が必須（空やプレースホルダは起動時に拒否）。生成例: `openssl rand -hex 32`。

!!! warning "production の起動ガード"
    `production` では `SECRET_KEY` が未設定/プレースホルダのまま、または `DATABASE_URL` が既定の `postgres:postgres@localhost` のままだと、設定バリデーションでエラーになり起動できません。実際の本番値を設定してください。

## 切り替え方法

`init` 時にプロファイルを指定して `.env` を生成し、`migrate` → `serve` の順で起動します。

```bash
context-hub init --profile personal   # または production
context-hub migrate
context-hub serve
```

実行時に環境変数で選ぶこともできます（`.env` の値より環境変数が優先されます）。

```bash
CH_PROFILE=personal context-hub serve
```

## 個別フィールドの上書き

プロファイルは各設定の**既定値**を決めるだけで、どのフィールドも対応する環境変数で個別に上書きできます（環境変数は大文字小文字を区別しません）。

```bash
# personal プロファイルのまま、取り込みだけ live にする
CH_PROFILE=personal INGEST_MODE=live context-hub ingest all
```

```bash
# .env で個別上書きする例
CH_PROFILE=personal
EMBEDDING_PROVIDER=bge-m3
LLM_PROVIDER=ollama
OLLAMA_BASE_URL=http://localhost:11434
```

!!! tip "現在の構成を確認する"
    どのプロファイル・取り込みモード・スケジューラで動いているかは、[設定GUI（/admin）](admin-gui.md) の **Status タブ**、または `GET /api/v1/status`（[REST / MCP API](api.md) 参照）で確認できます。ベクトル検索が無効化されていないか（`ftsDegraded`）もここで分かります。
