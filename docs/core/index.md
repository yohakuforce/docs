# core 概要

**@yohakuforce/core** は、Salesforce のメタデータを SQLite の知識グラフに変換し、そこから設計書（Markdown / HTML）を**決定的に**生成する CLI（コマンド名 `yohaku`）です。同じ入力からは必ず同じ出力が得られ、すべての事実に出典（provenance）が紐づきます。core 自身は LLM を一切呼び出しません。AI が踏み外さないための「足場（scaffold）」として動く、yohakuforce スイートの土台レイヤです。

---

## 何を解決するか

Salesforce の現場では、仕様書が更新されず属人化が進み、レビューやリリース準備のたびにゼロから全体像を把握し直す、という構造的な負担が積み重なります。

AI に設計書を書かせれば速い一方で、生成のたびに文章が揺れ、ときには事実まで変わってしまうという別の問題が生まれます。これではレビューにも監査にも使えません。

core はこの 2 つの問題を、関心を分離することで同時に解きます。

- **決定的な半分**（抽出 → 知識グラフ → レンダリング）を core が担う
- **解釈的な半分**（業務的な意味づけ・懸念点の説明）を、グラフを参照する AI ツールに委ねる

結果として、生成物は再現可能で監査でき、しかも AI による解説で補強された設計書になります。

!!! note "決定的（deterministic）とは"
    同じ `force-app/` メタデータから `yohaku render` を何度実行しても、生成される Markdown / HTML は完全に同一になります。これはゴールデンテストで担保されています。差分が出るのはソースが変わったときだけなので、生成物の diff がそのまま「仕様変更の diff」になります。

---

## 3 層分離 — core は LLM を呼ばない

yohakuforce スイートは、責務を 3 つの層に厳格に分けています。core はその最下層、**決定的処理層**です。

| 層 | 担当 | 思想 |
|---|---|---|
| 決定的処理層 | **core**（`yohaku` CLI / TypeScript） | 同じ入力には必ず同じ出力。再現性が命。**LLM を呼ばない。** |
| AI エージェント層 | Claude Code / Antigravity など | 曖昧さと判断は AI に委ねる。生データは読ませない。 |
| 人手補完層 | 人間 | 業務意図・顧客固有ルール・最終承認。AI が侵さない聖域。 |

この分離が core の設計の核です。core はメタデータを読み取り、グラフ化し、設計書の「骨格」を決定的に組み立てるところまでを担当します。骨格の中に空けられた `ai_managed` ブロック（業務的な意味づけや懸念点）は、AI ツールが後から埋めます。core はその穴に AI の出力を**安全に流し込む**仕組み（`html-write` / `explain-write`）だけを提供し、AI 推論そのものには関与しません。

!!! tip "なぜ LLM を core に入れないのか"
    LLM を決定的処理層に混ぜると、出力が揺れて再現性が壊れます。core を「揺れない足場」に保つことで、AI 側がいくら自由に推論しても、土台のメタデータ事実は常に一貫します。

---

## 知識グラフ → 設計書 の流れ

core のパイプラインは一方向です。

```
Salesforce メタデータ  →  SQLite 知識グラフ  →  決定的レンダリング  →  設計書（Markdown / HTML）
  (force-app/ / DX-MCP / org)   (yohaku graph build)      (yohaku render)
```

1. **抽出 + グラフ化** — `yohaku graph build` が `force-app/` のメタデータ（SObject / Field / Flow / ApexClass / ApexTrigger / ValidationRule / PermissionSet など）を解析し、`.yohaku/graph.sqlite` に知識グラフとして格納します。グラフは SQL で直接クエリできます（`yohaku graph query`）。
2. **レンダリング** — `yohaku render` がグラフを読み、設計書を生成します。`--format md` は AI が読む Markdown、`--format html` は人間がレビューする HTML 詳細設計書です（両方同時生成も可）。
3. **プレビュー** — `yohaku serve --watch` でローカルサーバを立て、ソース変更時に自動 rebuild + ブラウザ自動 reload しながら確認できます。

!!! warning "force-app/ は読むだけ、書かない"
    core はメタデータの正本である `force-app/` を**決して書き換えません**。グラフも設計書も、すべて `force-app/` を入力とする再生成物です。正本（コミット対象）と再生成物（gitignore 対象）の切り分けは [.yohaku 設定](configuration.md) を参照してください。

詳しい全コマンドは [CLI リファレンス](cli-reference.md)、HTML 詳細設計書の生成フローは [設計書パイプライン](design-docs.md) を参照してください。

---

## context-hub との関係

core は単体で完結しますが、**opt-in** で [context-hub](../context-hub/index.md) からプロジェクト／顧客コンテキストを取り込み、AI による意味づけを強化できます。

- 既定は `none`（context-hub 連携なし）。従来どおり core 単体で動きます。
- `.yohaku/config.json` の `contextProvider` を context-hub（共有 context-hub の MCP サーバ）に向けると、`yohaku context --kind <種別> --fqn <名前>` が、対象に関連する**抽象化済みコンテキスト**を `ContextBrief`（JSON）として返します。
- 取得されるのは抽象化された情報のみで、顧客名・PII・シークレットが生成物に書き込まれることはありません。

設定方法は [.yohaku 設定 / contextProvider](configuration.md) を参照してください。

```bash
# context-hub から Account に関連するコンテキストを取得（要 .yohaku/config.json 設定）
yohaku context --kind object --fqn Account
```

!!! note "スイート全体の中での core"
    core は 3 製品（**core** / **context-hub** / **ai-project-manager**）の最下層です。context-hub は共有コンテキストの供給源、ai-project-manager は運用タスクの自動化を担います。3 製品の関係は [3 製品の全体像](../getting-started/overview.md) を参照してください。

---

## 次のステップ

- まず導入する → [インストール / アップデート](install.md)
- コマンドを調べる → [CLI リファレンス](cli-reference.md)
- 設定ファイルを理解する → [.yohaku 設定](configuration.md)
- HTML 設計書を作る → [設計書パイプライン](design-docs.md)
