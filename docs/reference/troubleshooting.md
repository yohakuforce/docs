# トラブルシューティング

よくあるつまずきと解決法をまとめます。製品別の詳細は各リファレンスも参照してください。

---

## 共通 / 環境

| 症状 | 原因 / 対処 |
|---|---|
| `pip: command not found` | このマシンに `pip` 単体が無い。`pipx` を使うか、`python -m pip` / venv 内の `.venv/bin/pip` を使う |
| `npm: command not found` | Node.js の PATH 未設定。Node 20+ を導入し PATH を通す |
| `python: command not found` | `python3` を使う。pipx 経由なら不要 |

---

## core

| 症状 | 原因 / 対処 |
|---|---|
| `yohaku --version` が古い | `npm install -g @yohakuforce/core@latest` で更新 |
| `graph build` の結果が古い | `graph.sqlite` は再生成物。`yohaku graph build` で作り直す（コミット不要） |
| 設計書に LLM 充填部分が空 | `yohaku explain-prompts` → Claude Code で充填 → `yohaku html-write`。core 自身は LLM を呼ばない |
| context-hub の文脈が反映されない | `.yohaku/config.json` の `contextProvider` が共有 context-hub の URL を指しているか確認 |

詳細: [core CLI リファレンス](../core/cli-reference.md) / [設定](../core/configuration.md)

---

## context-hub

| 症状 | 原因 / 対処 |
|---|---|
| `/admin` が 401（GUI が読めない） | 起動を **`context-hub serve`** にする（生 uvicorn は `.env` を読まない）。GUI に DEV_API_KEY を貼ったか確認 |
| DEV_API_KEY が分からない | `Select-String DEV_API_KEY .env`（Win）/ `grep DEV_API_KEY .env`（Mac）。無ければ `context-hub init` が再発行 |
| Windows で意味検索が効かない | 標準 Python は `sqlite-vec` 拡張を読めず FTS-only に自動縮退。conda/miniforge Python か `production`（pgvector）を使う |
| 取り込みが空 | `INGEST_MODE=live` か、対象ソースが有効か（Sources タブ）。トークン未設定だと mock 扱い |
| production で DEV_API_KEY が効かない | 仕様。production では無効。発行済みコンシューマキー（ADMIN）を使う |

詳細: [設定GUI](../context-hub/admin-gui.md) / [データ取り込み](../context-hub/ingest.md)

---

## ai-project-manager

| 症状 | 原因 / 対処 |
|---|---|
| context-hub に繋がらない | `CONTEXT_HUB_API_KEY`（ai-pm）＝ `DEV_API_KEY`（context-hub）が**一致**しているか。`CONTEXT_HUB_BASE_URL` がコンテナから見て `host.docker.internal` か |
| 朝会・総括で LLM エラー | `LLM_PROVIDER` の選択を確認。まず `mock` で疎通 → 実 LLM へ。Docker 内では CLI 系は使えないことが多い（下記） |
| Docker 内で `claude-code`/`antigravity` が動かない | CLI を subprocess 起動する方式のため、コンテナ内に CLI が無いと不可。`local`（host のローカル LLM）か `mock`、または ai-pm をホスト直起動 |
| `docker compose up` が失敗 | `DB_PASSWORD` 未設定（compose が必須要求）。`.env` を確認 |
| 通知が溢れる | `NOTIFICATION_CHANNEL=local_file` で安全に開始。能力は 1〜2 個から |

詳細: [デプロイ](../ai-project-manager/deploy.md) / [LLM プロバイダ](../ai-project-manager/llm-providers.md)

---

## それでも解決しない場合

- 各リポの GitHub Issues:
    - context-hub: <https://github.com/yohakuforce/context-hub/issues>
    - core: <https://github.com/yohakuforce/core/issues>
- ログを添えて報告すると解決が速いです（context-hub は `serve` の標準出力、ai-pm は `docker compose logs`）。
