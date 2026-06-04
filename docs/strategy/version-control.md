# バージョン管理 / チーム共有

Salesforce 開発チームで「Salesforce ソース」「context-hub のコンテキスト」「ai-pm の実行履歴・結果」を
**必要なものだけ・無料・セキュアに**共有するための戦略です。

!!! abstract "結論（30秒サマリ）"
    **1 つの方法で全部管理しようとしない。データの性質で 2 層に分ける。**

    | 層 | 中身 | 性質 | 共有方法 |
    |---|---|---|---|
    | **A. 設計の正本** | Salesforce メタデータ + `.yohaku/` 設定 | テキスト・非機密・差分が意味を持つ | **Git（private リポ）** |
    | **B. 運用データ** | context-hub のコンテキスト / ai-pm の履歴・結果 | バイナリ DB・**顧客機密**・追記型 | **共有サービス 1 台に皆が接続** |

---

## なぜ「全部 Git」ではダメなのか

!!! danger "やってはいけない"
    context-hub / ai-pm の DB やダンプを Git（GitHub）に push しないこと。
    顧客機密が社外（GitHub のサーバ）に出てしまい、「raw data を第三者に出さない」という
    context-hub 自身の設計境界を破壊します。

1. **セキュリティ境界**: context-hub は顧客の Slack / Backlog / Redmine / 議事録を取り込む。
   これを GitHub の private リポに置いても「社外サーバに顧客データを保存」したことになる。
2. **Git はバイナリ DB に向かない**: `graph.sqlite` / Postgres ダンプはバイナリ。差分が取れず、
   リポが肥大化し、複数人の同時更新でマージ不能。
3. **可変・追記型データ**: 実行履歴・取込結果は日々増える「ログ」。Git の思想と噛み合わない。

→ **共有したいのは「ファイル」ではなく「最新状態への到達」**。それはバージョン管理ではなく
**共有サービス（皆が同じ 1 台に接続）**で実現します。

---

## 層 A：Git で管理するもの（Salesforce 開発リポ）

### :material-check: コミットする（正本）

- `force-app/`（Salesforce メタデータ）
- `.yohaku/config.json`（**`contextProvider` に共有 context-hub の URL** を書く ← 連携の肝）
- `.yohaku/context-map.yaml` / `.yohaku/secrets-rules.yaml`
- `docs/generated/`（core が生成する設計書 md/html。レビュー・共有用に有用）
- `sfdx-project.json` 等の DX 構成

### :material-close: コミットしない（再生成物 / 機密）

```gitignore
# yohakuforce core の再生成物（graph はメタデータから再ビルド可能）
.yohaku/graph.sqlite
.yohaku/graph.sqlite-*
.yohaku/cache/
.yohaku/metrics-cache/
.yohaku/coverage.json
.yohaku/onboarding-state.json
.yohaku/build.*
.yohaku/hook-timings.jsonl

# シークレット・環境
.env
.env.local
.env.*.local

# Salesforce / Node / OS
.sf/
.sfdx/
node_modules/
.DS_Store
*.log
```

!!! note "graph.sqlite は正本ではない"
    `graph.sqlite` は「メタデータから作る索引」。各自 `yohaku graph build` で再生成できるので
    コミットしません。`docs/generated/` は逆に「人がレビューする成果物」なのでコミットします
    （差分ノイズが気になるなら HTML は ignore して内部配信、md だけコミットも可）。

### 共有方法

- **GitHub の private リポ（無料・人数無制限）**。ブランチ + PR でレビュー、`yohaku diff` で設計差分を意味づけ。
- 任意で CI（GitHub Actions 無料枠）に `yohaku graph build && yohaku render` を回し設計書の最新性を保証。

---

## 層 B：共有サービスで管理するもの（コンテキスト・実行データ）

context-hub と ai-pm は **「オンプレに 1 台ずつ立て、チーム全員がそこに接続する」**。
ファイルを配り合うのではなく、**全員が同じ最新状態を見る**。

```
        ┌────────── 社内の共有ホスト1台（業務時間中起動でOK） ──────────┐
        │  context-hub serve (REST/MCP, :8000) ← 顧客データはここ │
        │  ai-project-manager (Docker, :8001)  ← 実行履歴・結果   │
        │  PostgreSQL（両者の永続化）                             │
        └────────────────────────────────────────────────────────┘
              ▲                                    ▲
    Tailscale/VPN(無料)で安全接続           ブラウザGUI / MCP
              │                                    │
     開発者A（Claude Code + core が         開発者B / PM
      共有 context-hub を参照）             （/admin, /settings）
```

- **ホスト**: 既存の社内マシン 1 台（業務時間中に起動している共有 PC など。専用の常時起動サーバは不要）。追加費用ゼロ。
    - context-hub: `--profile production`（Postgres）で `serve`。**LAN/VPN 内のみ公開**（インターネット直公開しない）。
    - ai-pm: `docker compose up`（Postgres 同梱）。
- **アクセス（無料・セキュア）**: **Tailscale 無料プラン** か社内 VPN でメッシュ接続。
- **認証**: context-hub の `DEV_API_KEY` / コンシューマキー、ai-pm の `app_secret_key`。**Git には絶対に入れない**。

### 連携の肝：Salesforce 開発 × 共有コンテキスト

1. 層 A の `.yohaku/config.json` の `contextProvider` に **共有 context-hub の URL** を書く（Git 管理）。
2. 各開発者の `yohaku` / Claude Code が、**同じ共有 context-hub の最新コンテキスト**を参照して設計・実装。
3. → **「設計の正本=Git」「コンテキスト=共有サービス」が config 1 行で連結**。両方共有しつつ機密はオンプレに留まる。

### 履歴・監査・durability（バックアップ ≠ バージョン管理）

- **ai-pm の監査ログ**: `AUDIT_LOG_DIR` を設定すると LLM 呼び出し等を記録（入力はハッシュ匿名化）。
- **定期バックアップ（無料）**: 共有ホストで cron/タスクスケジューラから
  context-hub(SQLite) は `data/context_hub.db` を暗号化コピー、Postgres は `pg_dump | gpg -c` を社内ストレージへ。
- **チームで読みたい結果サマリ**: ai-pm が生成する日報・総括（マスク済みテキスト）だけを、必要なら
  **別の「ナレッジ用 private リポ」に派生スナップショット**として置くのは可（生 DB は置かない）。

---

## 制約の充足

=== "セキュア"

    - 顧客データ（context-hub の DB）は**オンプレから出さない**。Git にもクラウドにも上げない。
    - `.env` / トークン / 鍵は **Git 管理外**。配布は 1Password 等の共有金庫か安全な手段で。
    - 共有サービスは**インターネット直公開しない**（VPN/Tailscale 内 or LAN のみ）。
    - マスキング（`secrets-rules.yaml` / GUI のマスク表示）を有効に保つ。

=== "無料"

    | 項目 | 手段 | 費用 |
    |---|---|---|
    | 層 A の Git | GitHub private リポ（人数無制限） | 0 |
    | CI（任意） | GitHub Actions 無料枠 | 0 |
    | 共有 context-hub / ai-pm | 既存社内マシン 1 台でセルフホスト | 0 |
    | セキュアなリモート接続 | Tailscale 無料 / 社内 VPN | 0 |
    | バックアップ | cron + `pg_dump`/コピー + gpg | 0 |

---

## まとめ（やること）

1. **層 A**: Salesforce DX + `.yohaku` 設定を GitHub private で管理。再生成物は `.gitignore`。
2. **層 B**: 社内の共有マシン 1 台（業務時間中起動で可）に context-hub + ai-pm を立て、Tailscale/VPN でチーム接続。
3. **連結**: `.yohaku/config.json` の `contextProvider` を共有 context-hub の URL に向ける。
4. **履歴/監査**: ai-pm の監査ログ + 暗号化バックアップ。共有したい結果はマスク済み要約のみ別リポに（任意）。
5. **鍵・機密**: 一切 Git に入れない。共有金庫 or 安全な手段で配布。
