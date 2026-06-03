# LLM プロバイダ

!!! abstract "このページの要約"
    AI-Project-Manager は LLM を `LLM_PROVIDER` で切り替えます。対応プロバイダは **`claude-code` / `codex` / `antigravity` / `local` / `ollama` / `mock`** で、**いずれも課金 API ではなくサブスク CLI かローカル LLM** です（既定は `claude-code`）。`claude`（Anthropic API 直叩き）だけが課金パスで、**非推奨・本番禁止**です。CLI 系（claude-code / codex / antigravity）は **サブスク契約済みの CLI を subprocess で呼び出す**ため、Docker コンテナ内で使うには CLI 同梱かホスト直起動が必要です。**コンテナ内で現実的なのは `local`（`host.docker.internal` 経由のローカル LLM）か `mock`** です。

---

## プロバイダ一覧

| `LLM_PROVIDER` | 種別 | 呼び出し方 | 課金 | 用途 |
|---|---|---|---|---|
| **`claude-code`** | サブスク CLI | `claude -p <prompt> --output-format json`（subprocess） | なし（サブスク範囲） | **既定・推奨** |
| **`codex`** | サブスク CLI | `codex -q <prompt>`（subprocess） | なし（サブスク範囲） | 代替の CLI |
| **`antigravity`** | サブスク CLI | `<antigravity> -p <prompt>`（subprocess、フラグ可変） | なし（サブスク範囲） | Antigravity をサブスク利用している場合 |
| **`local`** | ローカル HTTP | OpenAI 互換 `/v1/chat/completions` | なし（ローカル） | Ollama / LM Studio / vLLM / llama.cpp など |
| **`ollama`** | ローカル HTTP | Ollama native `/api/generate` | なし（ローカル・完全オフライン） | Ollama 限定で使いたい場合 |
| **`mock`** | モック | 固定応答（LLM 呼び出しなし） | なし | テスト・CI・決定性検証 |
| `claude` | 課金 API | Anthropic API 直接呼び出し | **あり（課金）** | **非推奨・本番禁止** |

!!! warning "課金 API は使わない方針"
    既定の `claude-code` を含む 6 プロバイダはすべて **課金 API ゼロ**（サブスク CLI またはローカル LLM）です。`claude` だけが Anthropic の **課金 API を直接叩く非推奨パス**で、本番では使用しないでください。`claude` を使う場合のみ `ANTHROPIC_API_KEY`（`sk-ant-…`）が必須になります。

---

## ★ Docker での運用注意

CLI 系プロバイダは **ホスト上に認証済みインストールされた CLI** を subprocess で呼び出します。Docker コンテナ内には既定でこれらの CLI が無いため、そのままでは動きません。コンテナ運用では下表の現実性を踏まえて選んでください。

| プロバイダ | Docker コンテナ内での現実性 | 必要条件 |
|---|---|---|
| `claude-code` | △ 要対応 | CLI をイメージに同梱し PATH に通す、またはホストで直起動して使う |
| `codex` | △ 要対応 | 同上（`codex` を同梱 or ホスト直起動。`CODEX_CLI_PATH` で明示も可） |
| `antigravity` | △ 要対応 | 同上（`antigravity` を同梱 or ホスト直起動。`ANTIGRAVITY_CLI_PATH` で明示も可） |
| **`local`** | ◎ 現実的 | ホストのローカル LLM を `LOCAL_LLM_BASE_URL=http://host.docker.internal:11434/v1` のように指す |
| `ollama` | ○ | `OLLAMA_BASE_URL=http://host.docker.internal:11434` でホストの Ollama を指す |
| **`mock`** | ◎ 現実的 | 外部依存なし。疎通・CI に最適 |

!!! tip "コンテナ内では local か mock が無難"
    Docker で動かすなら、**ホストのローカル LLM を `host.docker.internal` 経由で叩く `local`**（または `ollama`）か、外部依存のない **`mock`** が現実的です。CLI 系をコンテナで使う場合は、その CLI をイメージへ同梱し認証を通す追加作業が必要です。

---

## 設定項目

`/settings` GUI または `.env` で設定します。タイムアウトはいずれも秒単位です。

### CLI 系（サブスク）

| 設定キー | 既定 | 説明 |
|---|---|---|
| `CLAUDE_CODE_CLI_PATH` | 空（PATH 自動検出） | `claude` のフルパス。`which claude` の出力 |
| `CLAUDE_CODE_TIMEOUT_SECONDS` | `120` | Claude Code CLI 呼び出しのタイムアウト |
| `CODEX_CLI_PATH` | 空（PATH 自動検出） | `codex` のフルパス。`which codex` の出力 |
| `CODEX_TIMEOUT_SECONDS` | `120` | Codex CLI のタイムアウト |
| `ANTIGRAVITY_CLI_PATH` | 空（PATH 自動検出） | `antigravity` のフルパス |
| `ANTIGRAVITY_PROMPT_FLAG` | `-p` | プロンプトを渡すフラグ（CLI 仕様に合わせ調整） |
| `ANTIGRAVITY_TIMEOUT_SECONDS` | `120` | Antigravity CLI のタイムアウト |

CLI が見つからない場合は、PATH に通っているか・上記 `*_CLI_PATH` にフルパスを設定したかを確認してください。

### ローカル LLM

| 設定キー | 既定 | 説明 |
|---|---|---|
| `LOCAL_LLM_BASE_URL` | `http://localhost:11434/v1` | **OpenAI 互換**のベース URL。Ollama=`:11434/v1` / LM Studio=`:1234/v1` / vLLM 等 |
| `LOCAL_LLM_MODEL` | `llama3` | サーバに導入済みのモデル名 |
| `LOCAL_LLM_API_KEY` | 空 | 鍵を要求するサーバ向け（通常は空） |
| `LOCAL_LLM_TIMEOUT_SECONDS` | `180` | ローカル LLM のタイムアウト |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama **native** API のベース URL |
| `OLLAMA_MODEL` | `llama3` | `ollama list` の出力名 |
| `OLLAMA_TIMEOUT_SECONDS` | `180` | Ollama のタイムアウト |

!!! note "`local` は Ollama 限定ではない"
    `local` プロバイダは **OpenAI 互換の `/v1/chat/completions` を公開するあらゆるローカルサーバ**で動作します（Ollama / LM Studio / vLLM / llama.cpp server / text-generation-webui など）。Ollama 固有の `/api/generate` を使いたい場合や完全オフライン用途では `ollama` プロバイダを使います。

### 課金 API（非推奨）

| 設定キー | 既定 | 説明 |
|---|---|---|
| `ANTHROPIC_API_KEY` | 空 | `LLM_PROVIDER=claude` のときのみ必須（`sk-ant-…`）。**非推奨・本番禁止** |

---

## どれを選ぶか

- **既定で運用** → `claude-code`（サブスク範囲・課金なし）。ホストで CLI が使える環境向け。
- **Codex / Antigravity を契約している** → `codex` / `antigravity`（いずれもサブスク範囲）。
- **Docker コンテナ内・オフライン重視** → `local`（`host.docker.internal` 経由）または `ollama`。
- **テスト / CI / 決定性が欲しい** → `mock`（会議メモからのタスク抽出は走らない＝決定性）。

会議メモからのタスク自動抽出は `claude-code`（または `ollama` 等の実プロバイダ）で実抽出され、`mock` では抽出されません。データ境界の考え方は [概要](index.md) を、設定 UI は [設定 GUI](gui.md) を参照してください。
