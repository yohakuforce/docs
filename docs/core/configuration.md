# .yohaku 設定

core の設定はプロジェクトルートの `.yohaku/` ディレクトリに集約されます。主な設定ファイルは `config.json`（context-hub 連携など）、`context-map.yaml`（オンボーディングの読み順）、`secrets-rules.yaml`（マスキング規約）の 3 つです。あわせて、何が「正本=コミット対象」で何が「再生成物=gitignore 対象」かを整理します。

---

## .yohaku/ の中身

`yohaku init` を実行すると `.yohaku/` に足場が展開されます。代表的なファイルは次のとおりです。

| ファイル | 役割 | 種別 |
|---|---|---|
| `config.json` | core の設定（`contextProvider` 等） | 正本（コミット） |
| `context-map.yaml` | オンボーディングの読み順・ドメイン定義 | 正本（コミット） |
| `secrets-rules.yaml` | AI 送信前のマスキング規約 | 正本（コミット） |
| `domains.yaml` | 業務ドメイン分類（`yohaku domains init` で生成） | 正本（コミット） |
| `graph.sqlite` | 知識グラフ本体 | 再生成物（gitignore） |
| `hook-timings.jsonl` | フック実行時間のローカルログ | ローカル限定（gitignore） |
| `coverage.json` | 取り込んだテストカバレッジ | 運用データ |

---

## 正本（コミット） vs 再生成物（gitignore）

core の鉄則は **「`force-app/` は正本。それ以外の生成物は再生成可能」** です。グラフも設計書も、すべて `force-app/` メタデータを入力とする再生成物です。したがって、リポジトリにコミットすべきものとそうでないものを明確に分けます。

| 対象 | 分類 | 理由 |
|---|---|---|
| `force-app/`（メタデータ） | **正本** | システムの真実。core は読むだけで書き換えない |
| `.yohaku/config.json` | **正本（コミット）** | プロジェクト設定。チームで共有する |
| `.yohaku/context-map.yaml` | **正本（コミット）** | オンボーディング定義。手で育てる |
| `.yohaku/secrets-rules.yaml` | **正本（コミット）** | マスキング規約。手で育てる |
| `.yohaku/domains.yaml` | **正本（コミット）** | 業務ドメイン分類。手で調整する |
| `.yohaku/graph.sqlite` | **再生成物（gitignore）** | `graph build` で何度でも再生成できる中間正本 |
| `.yohaku/graph.sqlite-journal` / `-shm` / `-wal` | **再生成物（gitignore）** | SQLite の作業ファイル |
| `.yohaku/onboarding-state.json` | ローカル（gitignore） | ローカル進捗状態 |
| `.yohaku/hook-timings.jsonl` | ローカル（gitignore） | 実行時テレメトリ（ローカル限定） |
| `docs/generated/`（Markdown / HTML 設計書） | **再生成物** | `render` で再生成できる |

!!! warning "graph.sqlite はコミットしない"
    `.yohaku/graph.sqlite` は再生成物です。バイナリで diff も取りづらく、`yohaku graph build` でいつでも復元できるため、**`.gitignore` に入れてコミットしないのが原則**です。core 配布元の `.gitignore` でも `graph.sqlite` 系（`-journal` / `-shm` / `-wal`）と `cache/` は除外されています。

!!! tip "設計書（docs/generated）をコミットするか"
    設計書は再生成物ですが、「ソースを clone せずに GitHub 上で設計書を読みたい」「PR で設計書の diff をレビューしたい」場合はあえてコミットする運用もあります。決定的生成なので diff はそのまま仕様変更の diff になります。チームの方針として [バージョン管理 / チーム共有](../strategy/version-control.md) を参照してください。

推奨 `.gitignore`（プロジェクト側）の例:

```gitignore
# yohaku 知識グラフ（再生成物）
.yohaku/graph.sqlite
.yohaku/graph.sqlite-journal
.yohaku/graph.sqlite-shm
.yohaku/graph.sqlite-wal
.yohaku/cache/
.yohaku/onboarding-state.json
.yohaku/hook-timings.jsonl
```

---

## config.json — contextProvider（context-hub 連携）

`.yohaku/config.json` の `contextProvider` セクションで、AI による意味づけを強化するための context-hub 連携を設定します。**opt-in** で、既定（セクション無し / `kind: none`）では従来どおり core 単体で動きます。

### 連携しない場合（既定）

`contextProvider` を書かない、または `none` を指定すると、`yohaku context` は空の `ContextBrief` を返します。

```json
{
  "contextProvider": { "kind": "none" }
}
```

### 共有 context-hub に向ける場合

`kind` を `context-hub` にし、context-hub の MCP サーバ（stdio）を起動するコマンドと、参照するプロジェクト ID を指定します。

```json
{
  "contextProvider": {
    "kind": "context-hub",
    "command": "python",
    "args": ["-m", "context_hub.mcp.server"],
    "cwd": "/abs/path/to/Context-Hub",
    "projectId": "proj-001",
    "topK": 5
  }
}
```

| フィールド | 必須 | 説明 |
|---|---|---|
| `kind` | ✓ | `"context-hub"` または `"none"` |
| `command` | ✓ | MCP サーバを起動するコマンド（例: `python`） |
| `args` | – | コマンド引数の配列 |
| `cwd` | – | コマンドの作業ディレクトリ（context-hub のパス） |
| `projectId` | ✓ | context-hub 内部のプロジェクト ID |
| `topK` | – | 取得する関連断片の最大件数（既定 `5`） |

設定後、関連コンテキストを取得できます。

```bash
yohaku context --kind object --fqn Account
```

!!! warning "シークレットは config.json に書かない"
    API キーなどの秘密値は **`config.json` に書きません**。context-hub の MCP サーバは spawn 時に呼び出し元の環境変数（ambient env）を継承するため、シークレットは環境変数で渡します。設定ファイルに残すべきは接続情報（コマンド・projectId）だけです。

!!! note "データ境界"
    context-hub 連携は stdio（localhost）経由で同一マシン内に閉じます。取得結果はプロンプト材料として実行時に使われるだけで、core 側で永続化されません。返るのは**抽象化済みコンテキスト**のみで、顧客名・PII・シークレットが生成物に書き込まれることはありません。詳細は [context-hub](../context-hub/index.md) を参照してください。

---

## context-map.yaml — オンボーディングの読み順

`.yohaku/context-map.yaml` は、`yohaku onboard context --role <persona>` が参照する「ドメイン定義」と「ペルソナ別の読み順」を制御します。利用者プロジェクトの実ドメインに合わせて編集します。

```yaml
project:
  name: sample-project
  domains:
    - id: sales
      description: 営業関連 (商談・受注・顧客)
      primary_objects: [Account, Order__c]
      key_docs:
        - docs/generated/objects/Account.md
        - docs/human/business-notes/sales-process.md
    - id: finance
      description: 経理・与信・請求
      primary_objects: [Account, Invoice__c, Claim__c]
      key_docs:
        - docs/generated/objects/Invoice__c.md

personas:
  new_joiner:
    goal: 2 週間で主要ドメインを理解する
    read_order:
      - docs/generated/system-index.md
      - docs/human/business-notes/overview.md
      - "domains:*"            # 全ドメインを順次
    depth: summary_first
    primary_agent: onboarding-guide
  reviewer:
    goal: PR を適切にレビューする
    read_order:
      - docs/ai-augmented/change-summaries/latest.md
      - ops/registry/manual-steps-registry.md
    depth: detail_on_demand
    primary_agent: review-assistant
```

| キー | 説明 |
|---|---|
| `project.domains[].id` | ドメイン識別子 |
| `project.domains[].primary_objects` | そのドメインの中心 SObject |
| `project.domains[].key_docs` | 関連ドキュメントへのパス |
| `personas.<role>.read_order` | 読むべき順序（`domains:*` で全ドメイン順次） |
| `personas.<role>.depth` | 読みの深さ（`summary_first` / `detail_on_demand` など） |
| `personas.<role>.primary_agent` | 担当 AI エージェント名 |

ペルソナは `new_joiner` / `reviewer` / `release_manager` / `customer_facing` が標準です。

---

## secrets-rules.yaml — マスキング規約

`.yohaku/secrets-rules.yaml` は、メタデータが **AI に送信される前に**マスキングするルールを定義します。プロジェクトの実情に応じて `pattern` を追加・上書きします。

```yaml
version: 1
rules:
  - id: email-address
    description: メールアドレス (PII)
    pattern: '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'
    level: confidential
    mask: hash

  - id: salesforce-id-18
    description: Salesforce 18 文字 ID
    pattern: '\b00[A-Z0-9]{16}\b'
    level: internal
    mask: preserve

  - id: api-key-like
    description: API キーらしき長い英数字列
    pattern: '\b[A-Za-z0-9_-]{32,}\b'
    level: secret
    mask: redact
```

| フィールド | 説明 |
|---|---|
| `id` | ルール識別子 |
| `description` | 何を検出するかの説明 |
| `pattern` | マッチさせる正規表現 |
| `level` | 機密度（`internal` / `confidential` / `secret`） |
| `mask` | マスキング方式（下表） |

| `mask` | 挙動 |
|---|---|
| `preserve` | そのまま残す（分類のみ） |
| `hash` | ハッシュ化して置換（PII 向け） |
| `redact` | 完全に伏字化（シークレット向け） |

!!! tip "プロジェクト固有ルールを足す"
    顧客名や社内コードなど、プロジェクト固有の機密はファイル末尾に追記します。

    ```yaml
      - id: customer-name-acme
        description: 顧客 ACME 社の名称
        pattern: 'ACME株式会社|ACME Inc\.'
        level: confidential
        mask: redact
    ```

!!! note "マスキングは多層防御の一部"
    `secrets-rules.yaml` は「AI に生データを読ませない」という 3 層分離の原則を補強する仕組みです。そもそも core 自身は LLM を呼ばず、context-hub からは抽象化済みコンテキストしか受け取らない設計と組み合わせて、機密が外部へ漏れない多層防御を構成します。
