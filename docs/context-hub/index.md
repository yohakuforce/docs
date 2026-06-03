# Context-Hub 概要

!!! abstract "要約"
    **Context-Hub** は、顧客プロジェクトの文脈（Slack / Backlog / Redmine / Gmail / 議事録）を取り込み、手元の **SQLite または PostgreSQL** に蓄え、**MCP（Model Context Protocol）と REST API** の両方で AI に提供する文脈基盤です。Claude Desktop や Claude Code が直接つながり、プロジェクト固有の情報に基づいて回答します。**生データ（raw data）を第三者サービスに送らない**設計が中核で、`pipx install` だけで動き出します（Docker 不要・Postgres 不要・API キー不要）。CLI 名は `context-hub`、PyPI パッケージ名は `yohakuforce-context-hub`（現行 0.3.0）。

---

## 何を解決するか

AI に「このプロジェクトのことを分かったうえで答えてほしい」と思っても、Slack の議論・Backlog/Redmine の課題・議事録は別々の場所に散らばっています。これらを一つの問い合わせ先に集約し、AI から検索・参照できるようにするのが Context-Hub です。

- **散在する文脈の一元化** — Slack / Backlog / Redmine / Gmail / 議事録・メモを 1 つのローカルストアに集約します。
- **AI からの直接アクセス** — MCP ツールと REST API でそのまま AI に渡せます。コピペや手作業の貼り付けは不要です。
- **取りこぼさない要約** — 議事録は取り込み時にタスクを抽出して永続化するため、読み直すたびに結果がブレません。

## MCP ネイティブ

Context-Hub は **MCP を一級市民**として設計しています。stdio トランスポートで Claude Desktop / Claude Code に直接プラグインでき、`search_context` や `get_issues` など 7 種のツールを提供します。HTTP REST API は同じ `QueryService` を薄くラップしたもう一つの入り口で、MCP と REST は**同じビジネスロジックを共有**します。

```
┌─────────────────────────────────────────────────────┐
│                  Context-Hub プロセス                │
│   ┌──────────────┐    ┌──────────────────────────┐  │
│   │  MCP サーバ  │    │     FastAPI REST API      │  │
│   │  (stdio)     │    │  /api/v1/{projects,query} │  │
│   └──────┬───────┘    └───────────┬──────────────┘  │
│          └───────────┬────────────┘                  │
│                      ▼                               │
│               QueryService（共有）                   │
│          VectorStore + FTS + RRF フュージョン        │
│          ┌───────────┴───────────┐                   │
│          ▼                       ▼                   │
│    SQLite (sqlite-vec)    PostgreSQL (pgvector)       │
└─────────────────────────────────────────────────────┘
```

詳細は [REST / MCP API](api.md) を参照してください。連携先の AI エージェント本体については [AI-Project-Manager のデプロイ](../ai-project-manager/deploy.md) も合わせてご覧ください。

## ハイブリッド検索（FTS + ベクトル / RRF）

検索は **全文検索（FTS5）** と **ベクトル検索（sqlite-vec / pgvector）** を組み合わせ、両者の結果を **RRF（Reciprocal Rank Fusion）** で統合します。キーワード一致と意味的な近さの両方を拾えるため、「正確な単語」でも「言い回しが違う文章」でも引っかかります。

| 検索方式 | 強み | バックエンド |
|---|---|---|
| FTS5（全文検索） | 正確なキーワード一致・高速 | SQLite FTS5 |
| ベクトル検索 | 意味的な類似（言い換えに強い） | sqlite-vec / pgvector |
| RRF フュージョン | 両者の順位を統合し総合的に最良の結果を返す | QueryService 共有層 |

!!! note "埋め込みモデルとプロファイル"
    意味検索の品質は埋め込みモデルに依存します。`quickstart` プロファイルの既定はハッシュベースの **mock 埋め込み**（ゼロ依存・意味的な精度なし）で、`personal` / `production` では **BGE-M3** を使います。詳しくは [プロファイル](profiles.md) を参照してください。

!!! warning "Windows での自動デグレード"
    python.org 配布の Windows ビルドは `sqlite3` が拡張ロード非対応のため `sqlite-vec` を読み込めません。その場合 Context-Hub は自動的に **FTS-only モード**で動作します（取り込み・キーワード検索は動作、ベクトル検索のみ無効）。完全な意味検索が必要なら conda/miniforge の Python か `production`（PostgreSQL + pgvector）を使ってください。

## 「raw data を第三者に出さない」設計境界

Context-Hub の中核思想は、**取り込んだ生データを外部サービスへ送らない**ことです。

- **保存先はローカル** — データは手元の SQLite ファイル、または自分で管理する PostgreSQL に保存されます。
- **議事録 → タスク抽出は on-prem LLM** — 会議の生トランスクリプトを外部 API に投げず、`LLM_PROVIDER`（Ollama / Claude Code 等）で抽出します。抽出結果のみを永続化します。
- **Gmail はラベルベースのオプトイン** — 既定クエリ `label:context-hub` により、自分でタグ付けしたメールだけが取り込まれ、私的なメールはストアに入りません。
- **Admin GUI は localhost 前提** — `/admin` は資格情報を読み書きするため、`127.0.0.1` バインドでの利用を前提とし、すべてのデータ呼び出しに ADMIN API キーを要求します。

!!! tip "次に読む"
    まずは [インストール](install.md) でセットアップし、[プロファイル](profiles.md) で用途に合った構成を選び、[データ取り込み](ingest.md) で各ソースを繋いでください。設定はコマンドではなく [設定GUI（/admin）](admin-gui.md) からも行えます。
