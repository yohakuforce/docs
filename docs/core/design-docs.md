# 設計書パイプライン

core の目玉機能、HTML 詳細設計書パイプラインの解説です。同じ知識グラフから「AI が読む Markdown」と「人間がレビューする HTML 詳細設計書」を二系統で生成し、ブラウザでプレビューし、LLM に空ブロックを充填させるところまでを一気通貫で行います。v0.6.0 では HTML を**詳細設計レベル**（処理詳細・項目値の割り当て・計算方法）まで踏み込ませ、数式・入力規則の自然語化、フローチャートの日本語化、凡例ページを追加しました。

---

## 全体像

```
知識グラフ (graph.sqlite)
   ├─ yohaku render --format md      →  Markdown 設計書（AI が読む）
   └─ yohaku render --format html    →  HTML 詳細設計書（人間がレビューする）
                                         │
                                         ├─ yohaku serve --watch  →  ブラウザでプレビュー
                                         │
                                         └─ ai_managed ブロック（空）
                                              │
                                    yohaku explain-prompts  →  prompt+context (JSON)
                                              │   （LLM が充填: サブスクの Claude Code 等）
                                    yohaku html-write       →  充填内容を書き戻し
```

ポイントは、**決定的な骨格を core が作り、解釈的な中身を LLM が埋める**という分業です。core は LLM を呼ばず、骨格と「安全な書き戻し」だけを担当します。

---

## 1. Markdown と HTML を同時生成

`yohaku render` は出力形式を選べます。`--format md,html` で両方を一度に生成できます。

```bash
# Markdown と HTML を同時生成
yohaku render --format md,html
```

- **Markdown**（`--format md`、既定）: AI エージェントが読む前提の設計書。`docs/generated/` 配下。
- **HTML**（`--format html`）: 人間がレビューする詳細設計書。`docs/generated/html/` 配下。`index.html` がホームです。

生成される HTML は Apex / Trigger / LWC / Object / Flow の **component leaf** ページ、Cmd+K グローバル検索、業務フロー俯瞰タブ、Mermaid / HTML フォールバック描画を備えます。

!!! note "二系統にする理由"
    AI と人間では最適な媒体が違います。AI には構造化された Markdown が読みやすく、人間にはナビゲーションや検索が効く HTML がレビューしやすい。同じグラフから両方を決定的に生成することで、どちらも常に同じ事実を指します。

---

## 2. ブラウザでプレビュー（serve --watch）

生成した HTML はローカルサーバで開けます。`--watch` を付けると、ソース変更時に graph + HTML を自動再生成し、SSE でブラウザを自動 reload します。

```bash
# HTML をプレビュー
yohaku serve --port 4000

# ソース変更を監視して自動更新（force-app を監視）
yohaku serve --port 4000 --watch
```

`--watch` 時の動作は次のとおりです。

1. `force-app`（`--watch-dir` で変更可）の変更を検知
2. `graph build --incremental` でグラフを差分更新
3. HTML を再生成
4. SSE 経由でブラウザを自動 reload

!!! tip "設計レビューのリズム"
    `yohaku serve --watch` を立てたままメタデータを編集すると、変更が即座に HTML へ反映されます。設計レビューや「この修正で仕様書がどう変わるか」の確認に向いています。

---

## 3. 凡例ページ・数式の自然語化（v0.6.0）

v0.6.0 で HTML 設計書はより「読める」ものになりました。

- **凡例ページ**（`legend.html`）: 図形・ステップバッジ・項目値区分・色分け（事実 / AI）を 1 枚にまとめたページ。全 component ページの右上からリンクされます。
- **数式の自然語化**: 数式項目や入力規則（ValidationRule）を、日本語の算出ロジックへ展開します（原文は折りたたみで併記）。
- **フローチャートの日本語化**: Mermaid / ツリーのノードを自然な日本語にします（例: `Account を取得` / `o を登録` / `lines を 1 件ずつ繰り返す`、矢印も はい/いいえ・繰返）。
- **詳細設計セクション群**: Object の「項目値の割り当て」「計算項目・入力規則」、Apex/Trigger の「処理詳細」「項目値の割り当て」（触れるオブジェクト別タブ、JS 不要の CSS タブ）。

!!! note "事実と AI 解釈を色で区別"
    HTML 設計書では、決定的に確定する「事実」と、LLM が埋めた「解釈・確認」を色分けして区別します。凡例ページでこの色分けを確認できます。曖昧な項目を決定的テーブルに無理に押し込まず、完全性保証スケルトン付きの LLM ブロックへ振り分けているのが v0.6.0 の設計です。

---

## 4. explain-prompts → html-write の LLM 充填往復

HTML 設計書には、業務的な意味づけや懸念点など、メタデータだけからは決まらない箇所が **空の `ai_managed` ブロック**として用意されています。これを LLM に埋めてもらうのが充填往復です。

### ステップ 1: prompt + context を生成

```bash
yohaku explain-prompts --output prompts.json
```

`explain-prompts` は、空の `ai_managed` ブロックを埋めるための **prompt と context** を JSON で一括出力します。種別やコンポーネントで絞り込めます。

```bash
yohaku explain-prompts --kind business-meaning --types apex --max-items 20 --output prompts.json
```

### ステップ 2: LLM が充填

`prompts.json` を LLM に渡し、各ブロックを埋めた結果（`fill.json`）を受け取ります。

!!! warning "充填するのは LLM、core ではない"
    この充填ステップは **サブスクの Claude Code などの LLM が担当**します。**core 自身は LLM / API を一切呼びません。** core が提供するのは「埋めるための材料を出す（explain-prompts）」と「埋めた結果を安全に書き戻す（html-write）」の決定的な両端だけです。これが 3 層分離を保ったまま AI の解釈力を取り込むための設計です。

### ステップ 3: 充填内容を書き戻す

```bash
# まず dry-run で対象を確認
yohaku html-write --input fill.json --dry-run

# 書き戻す
yohaku html-write --input fill.json
```

`html-write` は AI-managed ブロック（`<!-- yohaku:block kind="ai_managed" ... -->`）にだけ書き込み、決定的な事実ブロックには触れません。実行結果は `updated` / `missing_components` / `missing_blocks` / `rejected` の件数で報告されます。

!!! tip "render を再実行しても充填は消えない"
    充填済みの内容は、その後 `yohaku render` を再実行しても保持されます（AI-managed ブロック preservation）。事実（メタデータ由来）は再生成で最新化され、解釈（LLM 由来）は残る、という二層が両立します。

Markdown 側にも同様の仕組みがあり、`/yohaku-explain` skill 連携の `yohaku explain-write --kind <種別> --fqn <name> --input <blocks.json>` で AI_MANAGED ブロックを安全に上書きできます。

---

## 5. リリースレビュー HTML とカバレッジ統合

設計書パイプラインにはリリース運用向けの機能も同梱されています。

### 差分 HTML（リリースレビュー）

```bash
yohaku diff --from v0.5.0 --to HEAD --format html --output release-review.html
```

`yohaku diff --format html` は、2 つの git ref 間の変更を HTML で出力し、各変更ファイルから component leaf ページへリンクします。リリース前のレビューに使えます。

### テストカバレッジ統合

```bash
# sf apex run test の JSON を取り込む
yohaku coverage import --input coverage.json

# 低カバレッジ順にサマリ表示
yohaku coverage show
```

`yohaku coverage import` は `sf apex run test --code-coverage --result-format json` の出力を取り込み、設計書に統合します。

---

## まとめ：決定的な足場 + AI の解釈

| 工程 | 担当 | 決定的か |
|---|---|---|
| メタデータ抽出・グラフ化 | core (`graph build`) | 決定的 |
| 設計書の骨格生成（md / html） | core (`render`) | 決定的 |
| 空ブロックの prompt 生成 | core (`explain-prompts`) | 決定的 |
| ブロックの中身を埋める | **LLM**（Claude Code 等） | 解釈的 |
| 充填内容の書き戻し | core (`html-write` / `explain-write`) | 決定的 |
| 差分 HTML / カバレッジ | core (`diff` / `coverage`) | 決定的 |

core は決定的な工程をすべて担い、解釈が必要な 1 点だけを LLM に委ねます。これにより、設計書は再現可能・監査可能でありながら、AI の説明力で補強されたものになります。

関連コマンドの詳細は [CLI リファレンス](cli-reference.md)、設定ファイルは [.yohaku 設定](configuration.md)、context-hub 連携は [context-hub](../context-hub/index.md) を参照してください。
