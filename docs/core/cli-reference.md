# CLI リファレンス

`yohaku` CLI の全コマンドリファレンスです。コマンドごとに **用途 / 主な引数 / 実行例** を示します。コマンド名・引数・既定値はすべて `yohaku --help`（実装の `printHelp`）に基づいています。

ヘルプとバージョンはいつでも確認できます。

```bash
yohaku --help      # 全コマンド一覧（-h でも可）
yohaku --version   # バージョン番号のみ（= yohaku version）
```

!!! note "共通の既定値"
    多くのコマンドが以下の既定パスを使います。`--root <dir>` で基準ディレクトリを変更できます。

    | 対象 | 既定値 |
    |---|---|
    | 知識グラフ DB | `.yohaku/graph.sqlite`（`--db` で変更可） |
    | 生成物出力先 | `docs/generated`（`--output` で変更可） |
    | Salesforce API バージョン | `62.0`（`--api` で変更可） |
    | domains 定義 | `.yohaku/domains.yaml`（`--path` で変更可） |
    | カバレッジ | `.yohaku/coverage.json` |

---

## クイック運用コマンド

### `yohaku init`

**用途**: 利用者の Salesforce DX プロジェクトに `.yohaku/` の足場（設定・スキャフォールド）を展開します。`--bootstrap` を付けると、init に続けて graph build と render まで一気に実行します。

**主な引数**

| 引数 | 説明 | 既定値 |
|---|---|---|
| `--bootstrap` | init + graph build + render を一括実行 | （なし） |
| `--target <dir>` | 展開先ディレクトリ | カレント |
| `--profile minimal\|standard\|full` | 同梱スキャフォールドの規模 | `standard` |
| `--project-name <name>` | プロジェクト名（`--name` も可） | ディレクトリ名 |
| `--language ja\|en` | 生成物の言語 | `ja` |
| `--api <version>` | Salesforce API バージョン | `62.0` |
| `--segment enterprise\|smb\|vendor` | 利用者セグメント | `unspecified` |
| `--conflict skip\|overwrite\|rename` | 既存ファイル衝突時の挙動 | `skip` |

**実行例**

```bash
# 最小構成でブートストラップ（init + build + render）
yohaku init --bootstrap --profile minimal

# 標準構成・プロジェクト名と言語を指定
yohaku init --bootstrap --profile standard --project-name acme-sfdx --language ja
```

### `yohaku sync`

**用途**: 日常運用コマンド。`graph build --incremental` と `render` を 1 コマンドでまとめて実行し、設計書を最新化します。

**主な引数**

| 引数 | 説明 | 既定値 |
|---|---|---|
| `--full-rebuild` | 差分ではなく全再構築する | （差分） |
| `--quiet` | ログを抑止 | （なし） |
| `--output <dir>` | 生成物出力先 | `docs/generated` |
| `--api <version>` | API バージョン | `62.0` |

**実行例**

```bash
# 差分で最新化（日常運用）
yohaku sync

# 全再構築して静かに実行
yohaku sync --full-rebuild --quiet
```

!!! tip "init = 初回、sync = 毎回"
    初回だけ `yohaku init --bootstrap`、以降は `yohaku sync` を回す、という運用が基本リズムです。

---

## 知識グラフ

### `yohaku graph build`

**用途**: `force-app/` 等のメタデータを解析し、SQLite 知識グラフ（`.yohaku/graph.sqlite`）を構築します。

**主な引数**

| 引数 | 説明 | 既定値 |
|---|---|---|
| `--incremental` | 変更分のみ反映（差分ビルド） | （フルビルド） |
| `--source local\|dx-mcp\|org` | 入力ソース | `local` |
| `--types ApexClass,Flow,...` | 対象メタデータ種を絞り込む | 全種 |
| `--retrieve-to <dir>` | `--source=org` 取得先ディレクトリ | （内部） |
| `--async` | 子プロセスを detach し即座に戻る（Claude Code hook 向け） | （同期） |
| `--quiet` | ログ抑止 | （なし） |
| `--no-timing-log` | 実行時間ログ（`.yohaku/hook-timings.jsonl`）を抑止 | （記録する） |

**実行例**

```bash
# ローカルの force-app/ から差分ビルド
yohaku graph build --incremental

# 認証済み org（sf CLI の defaultusername）から package.xml 一括 retrieve
yohaku graph build --source org

# Apex と Flow だけを対象にビルド
yohaku graph build --types ApexClass,Flow
```

!!! note "--source=org"
    `--source=org` は sf CLI で認証済みの `defaultusername` から `package.xml` ベースで一括 retrieve します（Phase 4）。`--async` は hook から呼ぶときに有用で、実行時間は `.yohaku/hook-timings.jsonl` に記録されます。

### `yohaku graph query`

**用途**: 構築済みの知識グラフに対して任意の SQL を実行し、結果を出力します。AI / 人間が事実を直接確認するための窓口です。

**主な引数**: SQL 文字列を直接渡します。

**実行例**

```bash
yohaku graph query "SELECT name FROM sobject LIMIT 10"
yohaku graph query "SELECT name FROM apex_class WHERE name LIKE '%Controller'"
```

### `yohaku graph schema`

**用途**: グラフのスキーマを出力します。バリデーション用の meta schema と、実 SQLite テーブル定義（PRAGMA 代替・LLM 向け）の 2 系統があります。

**主な引数**

| 引数 | 説明 | 既定値 |
|---|---|---|
| `--format json\|markdown` | 出力形式 | `json` |
| `--tables` | 実 SQLite テーブル定義を出力 | （meta schema） |
| `--table <name>` | 特定テーブルのみ（`--tables` と併用） | 全テーブル |

**実行例**

```bash
# バリデーション用の meta schema を Markdown で
yohaku graph schema --format markdown

# 実テーブル定義（LLM 向け PRAGMA 代替）
yohaku graph schema --tables

# 特定テーブルだけ
yohaku graph schema --tables --table apex_class
```

---

## レンダリング

### `yohaku render`

**用途**: 知識グラフから設計書を決定的に生成します。引数なしは後方互換で `system-index + objects` を生成、`all` で全種を生成します。

**ターゲット**

| ターゲット | 生成物 |
|---|---|
| `render`（引数なし） | system-index + objects（後方互換） |
| `render all` | 全種を一括 |
| `render system-index` | プロジェクト全体像 |
| `render objects` | SObject 個別（field / ValidationRule / 依存関係） |
| `render flows` | Flow 個別 |
| `render apex` | ApexClass 個別 |
| `render triggers` | ApexTrigger 個別 |
| `render permissions` | PermissionSet + Profile |
| `render validation-rules` | ValidationRule 個別 |

**主な引数**

| 引数 | 説明 | 既定値 |
|---|---|---|
| `--output <dir>` | 出力先ディレクトリ | `docs/generated` |
| `--format md\|html\|md,html` | 出力形式 | `md`（後方互換） |

**実行例**

```bash
# 全種を Markdown で生成
yohaku render all

# プロジェクト全体像だけ
yohaku render system-index

# Markdown と HTML を同時生成（html は <output>/html/ に配置）
yohaku render --format md,html
```

!!! tip "HTML 詳細設計書を作りたいとき"
    `--format html` を使うと人間がレビューする HTML 詳細設計書が `<output>/html/` に生成されます。生成・プレビュー・LLM 充填の一連の流れは [設計書パイプライン](design-docs.md) を参照してください。

---

## 差分（Diff）

### `yohaku diff`

**用途**: 2 つの git ref 間でメタデータの差分を抽出します。JSON 出力のほか、リリースレビュー用の HTML 出力（各変更ファイルから component leaf へリンク）も生成できます。

**主な引数**

| 引数 | 説明 | 既定値 |
|---|---|---|
| `--from <ref>` | 比較元 ref（必須） | （なし） |
| `--to <ref>` | 比較先 ref | `HEAD` |
| `--json` | JSON で出力 | （なし） |
| `--path-prefix <prefix>` | 対象パスを絞り込む | （全体） |
| `--limit <n>` | 対象ファイル上限 | `1000` |
| `--include-static-analysis <sarif>` | SARIF 静的解析結果を取り込む | （なし） |
| `--format html` | HTML 出力 | （テキスト/JSON） |
| `--output <file>` | HTML 出力先 | （例: `release-review.html`） |

**実行例**

```bash
# 直前のタグから HEAD までの差分を JSON で
yohaku diff --from v0.5.0 --json

# リリースレビュー用 HTML を生成
yohaku diff --from v0.5.0 --to HEAD --format html --output release-review.html
```

---

## レンダリング（HTML 充填・LLM 連携）

### `yohaku explain-prompts`

**用途**: HTML の空 `ai_managed` ブロックを LLM に埋めてもらうための **prompt + context** を一括生成します。出力 JSON を LLM に渡し、結果（`fill.json`）を `html-write` で書き戻します。

**主な引数**

| 引数 | 説明 | 既定値 |
|---|---|---|
| `--kind business-meaning,concerns` | 充填対象のブロック種別 | 全種 |
| `--types apex,trigger,...` | コンポーネント種で絞り込む | 全種 |
| `--names X,Y,...` | コンポーネント名で絞り込む | 全件 |
| `--max-items <n>` | 出力アイテム上限 | （上限なし） |
| `--output <file>` | 出力先（省略時は標準出力） | stdout |

**実行例**

```bash
yohaku explain-prompts --output prompts.json
yohaku explain-prompts --kind business-meaning --types apex --max-items 20 --output prompts.json
```

### `yohaku html-write`

**用途**: LLM が埋めた内容（`fill.json`）を、HTML の AI-managed ブロック（`<!-- yohaku:block kind="ai_managed" ... -->`）にだけ安全に書き戻します。他のブロックには触れません。

**主な引数**

| 引数 | 説明 | 既定値 |
|---|---|---|
| `--input <fill.json>` | 充填内容 JSON（必須） | （なし） |
| `--out <dir>` | 生成物ルート | `docs/generated` |
| `--dry-run` | 書き込まず結果のみ表示 | （書き込む） |

**実行例**

```bash
# 充填内容を書き戻す
yohaku html-write --input fill.json

# まず dry-run で対象を確認
yohaku html-write --input fill.json --dry-run
```

!!! note "explain-prompts → html-write の往復"
    `explain-prompts` で prompt を出し、サブスクの Claude Code 等が充填し、`html-write` で書き戻す——この往復は LLM 側で完結します。**core 自身は API を呼びません。** 詳細は [設計書パイプライン](design-docs.md) を参照してください。

### `yohaku explain-write`

**用途**: Markdown 側（`/yohaku-explain` skill 連携）で、LLM が生成したブロックを `AI_MANAGED` ブロックにだけ安全に上書きします。

**主な引数**

| 引数 | 説明 |
|---|---|
| `--kind <種別>` | `apexClass` / `apexTrigger` / `flow` / `object` / `lwc` / `auraBundle` / `flexiPage` など |
| `--fqn <name>` | 対象コンポーネント名（必須） |
| `--input <blocks.json>` | `{ blockId: content, ... }` 形式の JSON（必須） |
| `--project-root <dir>` | プロジェクトルート |
| `--output-dir <dir>` | 出力先 |

**実行例**

```bash
yohaku explain-write --kind apexClass --fqn AccountService --input blocks.json
```

---

## 業務ドメイン（Domains）

### `yohaku domains init`

**用途**: 知識グラフからヒューリスティックで初期 `domains.yaml`（業務ドメイン分類）を生成します。

**主な引数**: `--path <yaml>`（既定 `.yohaku/domains.yaml`）/ `--force`（既存を上書き）

```bash
yohaku domains init
yohaku domains init --force
```

### `yohaku domains sync`

**用途**: グラフに追加された新メンバを、`domains.yaml` の `Unclassified` に追記します。

```bash
yohaku domains sync
```

### `yohaku domains lint`

**用途**: `domains.yaml` の重複 id / 多重所属 / 孤立メンバ / 未分類を検出します。エラーがあると終了コード 1 を返します。

```bash
yohaku domains lint
```

---

## オンボーディング

### `yohaku onboard context`

**用途**: ペルソナ別に「読むべき順序」を踏まえたオンボーディング用コンテキストを出力します。

**主な引数**: `--role <new_joiner|reviewer|release_manager|customer_facing>`

```bash
yohaku onboard context --role new_joiner
```

### `yohaku onboard state`

**用途**: オンボーディングの進捗状態を管理します。サブコマンドは `show` / `record-step` / `increment-questions` / `reset`。

```bash
yohaku onboard state show
yohaku onboard state record-step --role new_joiner --step intro --entities Account,Order__c
yohaku onboard state increment-questions --role reviewer
yohaku onboard state reset --role new_joiner
```

### `yohaku onboard faq`

**用途**: 対話ログ（Markdown）から FAQ を抽出します。

**主な引数**: `--input <dialog.md>`（必須）/ `--topic <name>`（既定 `general`）/ `--min-occurrences <n>`（既定 `1`）

```bash
yohaku onboard faq extract --input dialog.md --topic billing --min-occurrences 2
```

---

## ローカルプレビュー

### `yohaku serve`

**用途**: `docs/generated/html` を配信する軽量 HTTP サーバを立てます。`--watch` でソース変更時に graph + HTML を再生成し、SSE でブラウザを自動 reload します。

**主な引数**

| 引数 | 説明 | 既定値 |
|---|---|---|
| `--port <n>` | ポート | `4000` |
| `--host <addr>` | バインドアドレス | `127.0.0.1` |
| `--dir <dir>` | 配信ディレクトリ | `docs/generated/html` |
| `--watch` | 変更監視 + 自動 rebuild + 自動 reload | （なし） |
| `--watch-dir <dir>` | 監視対象ディレクトリ | `force-app` |
| `--quiet` | ログ抑止 | （なし） |

**実行例**

```bash
# HTML をプレビュー
yohaku serve --port 4000

# ソース変更を監視して自動更新
yohaku serve --port 4000 --watch
```

!!! warning "0.0.0.0 へのバインド"
    `--host 0.0.0.0` を指定すると LAN からアクセス可能になります（起動時に警告が出ます）。共有が不要なら既定の `127.0.0.1` のままにしてください。

---

## テストカバレッジ

### `yohaku coverage import`

**用途**: `sf apex run test --code-coverage --result-format json` の出力を取り込み、正規化して保存します。

**主な引数**: `--input <coverage.json>`（必須）/ `--out <path>`（既定 `.yohaku/coverage.json`）

```bash
yohaku coverage import --input coverage.json
```

### `yohaku coverage show`

**用途**: 取り込み済みカバレッジのサマリを、低カバレッジ順に上位 10 件表示します。

**主な引数**: `--path <path>`（既定 `.yohaku/coverage.json`）

```bash
yohaku coverage show
```

---

## その他

### `yohaku validate`

**用途**: グラフ JSON を meta schema で検証します。

**主な引数**: `--target <graph.json>`

```bash
yohaku validate --target graph.json
```

### `yohaku context`

**用途**: opt-in の context-hub 連携。対象に関連するプロジェクト／顧客コンテキストを context-hub から取得し、`ContextBrief`（JSON）で出力します。`.yohaku/config.json` の `contextProvider` を参照し、未設定なら空 brief を返します（従来動作のまま）。

**主な引数**: `--kind <metadataKind>`（必須）/ `--fqn <name>`（必須）/ `--project-root <dir>`

```bash
yohaku context --kind object --fqn Account
```

詳細は [.yohaku 設定 / contextProvider](configuration.md) と [context-hub 連携](../context-hub/index.md) を参照してください。

### `yohaku metrics`

**用途**: LLM トークン使用量などのメトリクスを記録・表示します。

**主な引数**

- `metrics show --period day|week|month|all`（既定 `month`）
- `metrics record --model <id> --command <name> --in <tokens> --out <tokens>`

```bash
yohaku metrics show --period week
yohaku metrics record --model claude-sonnet --command explain-prompts --in 1200 --out 800
```

### `yohaku version`

**用途**: バージョン番号を出力します（`yohaku --version` と同等）。

```bash
yohaku version
```

---

## 終了コードの目安

| コード | 意味 |
|---|---|
| `0` | 正常終了 |
| `1` | 実行時エラー（処理失敗・lint エラー検出など） |
| `2` | 引数不正・前提ファイル欠如（例: グラフ未構築、未知のサブコマンド） |

!!! note "前提ファイルの自動チェック"
    多くのコマンドは `.yohaku/graph.sqlite` の存在を前提にし、無い場合は `Run "yohaku graph build" first.` と案内して終了コード 2 を返します。`domains` / `coverage` 系も同様に、前段コマンドの実行を促します。
