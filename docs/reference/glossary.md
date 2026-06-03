# 用語集

| 用語 | 意味 |
|---|---|
| **core** | `@yohakuforce/core`。Salesforce メタデータを知識グラフ化し設計書を生成する CLI（`yohaku`）。 |
| **context-hub** | 顧客データを取り込み AI に文脈提供するローカル基盤。PyPI `yohakuforce-context-hub`。 |
| **ai-project-manager（ai-pm）** | context-hub の文脈で進行管理する AI。private + Docker。 |
| **知識グラフ（graph.sqlite）** | core が Salesforce メタデータから生成する索引。再生成可能なため Git 管理しない。 |
| **設計書パイプライン** | `yohaku render` で md/html 設計書を生成する流れ。v0.6.0 で詳細設計レベルに到達。 |
| **contextProvider** | core の `.yohaku/config.json` の設定。共有 context-hub の URL を指すと文脈を参照できる。 |
| **MCP（Model Context Protocol）** | AI クライアント（Claude Desktop / Code）が文脈源に接続する標準。context-hub が対応。 |
| **DEV_API_KEY** | context-hub の開発時 ADMIN キー。`init` が自動発行。`/admin` と ai-pm の接続に使う。 |
| **コンシューマキー** | context-hub の production 認証キー。DEV_API_KEY が無効な本番で使う。 |
| **プロファイル** | context-hub の構成プリセット（quickstart / personal / production）。 |
| **ハイブリッド検索** | 全文検索（FTS）とベクトル検索を RRF で統合する context-hub の検索方式。 |
| **camelCase 契約** | context-hub 0.3.0 で REST 応答キーを camelCase に統一した破壊的変更。 |
| **7 つの能力** | ai-pm の Plan / Assign / Track / Alert / Overview / Standup / WrapUp。 |
| **CANONICAL_STEP_ORDER** | ai-pm スケジューラの固定実行順。時刻は調整可だが順序は不変。 |
| **リーダー確認ゲート** | ai-pm が全体ステータス分析（final_analysis）を人の確認操作で発火させる仕組み。 |
| **LLM プロバイダ** | ai-pm の LLM 接続先（claude-code/codex/antigravity/local/ollama/mock）。既定はサブスクの claude-code。 |
| **データ境界** | 顧客の生データを外部 API/SaaS に出さない、3 製品共通の設計原則。 |
| **層 A / 層 B** | 本ヘルプの戦略用語。A=Git で管理する設計の正本、B=共有サービスで持つ運用データ。 |
