# AI ハーネス連携（Claude Code / Codex / Antigravity / ローカル LLM）

!!! abstract "要約"
    `yohaku` は **Claude Code / OpenAI Codex / Google Antigravity / ローカル LLM** のいずれからでも使えます。共通の前提は「**CLI が唯一のソース**」——どのツールでも `yohaku render --format md,html` 等が決定的に同じ出力を出します。スラッシュコマンドを持つのは Claude Code だけで、他は `AGENTS.md` やチートシート経由で CLI を直接呼びます。`yohaku init --bootstrap` が各ツール向けの導線（`.claude/` / `AGENTS.md` / `prompts/`）を配置します。

---

## Claude Code（ネイティブ統合）

スラッシュコマンドと専門サブエージェントが `.claude/` に配置され、会話的に駆動できます（裏で CLI と知識グラフを呼ぶ）。配置されるコマンドは `--profile` で変わります。

### 理解・分析・リリース

| スキル | 引数・オプション | 役割 |
|---|---|---|
| `/onboard` | `[--role new_joiner\|reviewer\|release_manager\|customer_facing]` | persona 別に対話的オンボーディング |
| `/explain` | `<object\|field\|flow\|apex\|permissionSet 名>` | 特定エンティティを多角的に詳細説明 |
| `/impact` | `<object\|field\|flow\|apex 名>` | 変更時の影響範囲を表示 |
| `/classify-diff` | `[--from <ref>] [--to <ref>] [--include-static-analysis <sarif>]` | Git 差分を 7 分類で意味づけ |
| `/change-summary` | `[--from <ref>] [--to <ref>] [--output <dir>]` | PR レビュー用 change-summary を生成・配置 |
| `/release-prep` | `--from <prev-tag> [--to <ref>] [--version <vX.Y.Z>] [--include-static-analysis <sarif>]` | リリース資材を 6 セクションで生成 |
| `/manual-steps` | `[--release <ver>] [--status pending\|done] [--category pre_release\|during_release\|post_release]` | 手動作業レジストリを横断参照 |
| `/analyze-batch-limits` | `<ObjectApiName>` | トリガ処理全体のクエリ数・バッチ許容件数を算出 |

### HTML 設計書 / ドメイン / カバレッジ / AI 追記

| スキル | 引数・オプション | 役割 |
|---|---|---|
| `/yohaku-html-build` | `[--source local\|org] [--types apex,trigger,…] [--strict]` | HTML 設計書を一括生成 |
| `/yohaku-serve` | `[--port 4000] [--watch] [--dir docs/generated/html]` | ローカル配信（`--watch` で自動 reload） |
| `/yohaku-diff-html` | `--from <ref> [--to <ref>] [--output release-review.html]` | 2 リビジョン間を Release Review HTML 化 |
| `/yohaku-explain-prompts` | `[--kind business-meaning,concerns] [--types apex,object] [--output prompts.json]` | AI ブロック充填用 prompt を一括生成 |
| `/yohaku-explain` | `<kind> <fqn> [block-id…]` | AI_MANAGED ブロックを再生成して書き戻し |
| `/yohaku-html-write` | `--input fill.json [--dry-run]` | HTML の AI ブロックに LLM 回答を安全反映 |
| `/yohaku-domains` | `init \| sync \| lint` | `domains.yaml` を生成 / 追記 / 検査 |
| `/yohaku-coverage-import` | `import --input coverage.json \| show` | Apex カバレッジを設計書に取り込み |

### 専門サブエージェント

スキルは内部で、用途特化のサブエージェント群（`.claude/agents/`）を呼び分けます。いずれも知識グラフと決定的ファクトのみを根拠に動きます。

- **グラフ照会**: `graph-querier` / `apex-query-tracer` / `flow-query-tracer` / `cascade-tracer` / `batch-calculator`
- **差分分類**: `automation-classifier` / `data-model-classifier` / `logic-classifier` / `permission-classifier` / `ui-classifier`
- **文書生成**: `object-documenter` / `explain-writer`
- **オンボーディング**: `onboarding-guide` / `customer-impact-explainer` / `manual-step-extractor`
- **リリース**: `release-advisor` / `release-composer` / `rollback-drafter` / `review-assistant`

### profile による配置の違い

| profile | 配置されるスキル |
|---|---|
| `minimal` | `/onboard` `/explain` `/impact` |
| `standard` | ＋ `/classify-diff` `/change-summary` ＋ HTML・ドメイン系 |
| `full` | ＋ `/release-prep` `/manual-steps` ＋ `/yohaku-diff-html` `/yohaku-coverage-import` |

!!! note "導入"
    `yohaku init --bootstrap --profile <minimal|standard|full>` でスキル・サブエージェント・hooks がプロジェクトに配置されます。Claude Code を開き、まず `/onboard` から始めてください。

---

## Codex / Antigravity（AGENTS.md 方式）

スラッシュコマンドはありません。`prompts/codex/AGENTS-snippet.md` / `prompts/antigravity/AGENTS-snippet.md` を root の `AGENTS.md` に取り込むと、「生 XML / メタデータを grep せず yohaku CLI を呼ぶ」よう誘導されます。

| ユーザーの問い | 提示するコマンド |
|---|---|
| このオブジェクトを変えたら何が壊れる？ | `yohaku impact SObject:<name>` |
| このクラスはどこから呼ばれている？ | `yohaku graph query "…"` |
| 設計書を人にレビューしてもらいたい | `yohaku render --format md,html` |
| 業務的意味を AI に埋めてほしい | `yohaku explain-prompts --output prompts.json` → 回答を `fill.json` 化 → `yohaku html-write --input fill.json` |
| このリリースの変更点をレビューしたい | `yohaku diff --from <ref> --to HEAD --format html` |

役割分担は **トポロジー（どこ・どう繋がる）= CLI** / **セマンティクス（何をしている）= ソース直読**。自律ループで `render` を繰り返しても AI-managed ブロックの充填内容は保持されます。

---

## ローカル LLM

専用テンプレートは不要で、**シェルを実行できれば動きます**（`qwen2.5-coder` 等の小型 coder 系でも可）。`yohaku --help` でコマンドを発見させ、必要な 1 行だけ渡すと安定します。LLM が介在するのは `explain-prompts` → `html-write` の間だけなので、`--dry-run` で `rejected` が 0 件かを必ず確認してください。

!!! tip "共通原則"
    どのハーネスでも、**決定的な事実は CLI から取り、解釈だけを LLM に委ねる**のが安全です。core 自身は LLM を呼ばず、生成物の出所（provenance）を保証します（[設計書パイプライン](design-docs.md)）。
