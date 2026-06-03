# FAQ

## 全般

??? question "3 製品はどれから入れればいい？"
    **core 単独**から始めるのがおすすめです（独立して動き、撤退コストもほぼゼロ）。
    文脈共有が要るなら context-hub、進行管理の自動化が要るなら ai-project-manager を足します。
    導入順は「context-hub →ai-project-manager」。詳しくは [導入戦略](../strategy/adoption.md)。

??? question "API 課金は発生しますか？"
    いいえ。3 製品とも **既定では課金 API を使いません**。

    - core は LLM を呼ばず決定的に生成（充填は利用者の Claude Code = サブスク）。
    - context-hub は課金プロバイダ（`openai`/`claude`）を**明示的に拒否**し、`claude-code`/`ollama` 等で動く。
    - ai-pm の既定は `claude-code`（サブスク）。`claude`（課金 API）は非推奨の opt-in のみ。

??? question "顧客データはどこに保存されますか？外部に出ますか？"
    context-hub のローカル DB（オンプレ）に保存され、**外部 API / SaaS に生データは出ません**。
    これは 3 製品共通の設計境界です。チーム共有も「オンプレの共有サービスに接続」で行い、
    Git やクラウドには載せません（[戦略](../strategy/version-control.md)）。

## バージョン管理

??? question "context-hub のコンテキストを Git で共有してもいい？"
    いいえ。顧客機密を含む DB を Git（GitHub）に置くと境界を破壊します。しかもバイナリでマージ不能。
    **共有サービス 1 台に皆が接続**する方式にしてください（[戦略](../strategy/version-control.md)）。

??? question "core の `graph.sqlite` はコミットすべき？"
    いいえ。メタデータから `yohaku graph build` で再生成できる「索引」なので `.gitignore` します。
    一方 `docs/generated/`（設計書）はレビュー共有用にコミットします。

??? question "実行履歴を残したい。Git に入れる？"
    Git ではなく、ai-pm の監査ログ（`AUDIT_LOG_DIR`）+ 暗号化バックアップ（`pg_dump | gpg`）で残します。
    チームで読みたい結果サマリ（マスク済み）だけ、必要なら別のナレッジ用リポに置くのは可。

## LLM

??? question "Claude Code 以外の LLM は使える？"
    使えます。ai-pm は `claude-code` / `codex` / `antigravity` / `local`（OpenAI 互換ローカル）/ `ollama` /
    `mock` に対応。`local` は Ollama に限らず LM Studio・vLLM・llama.cpp 等で動きます。
    詳細: [LLM プロバイダ](../ai-project-manager/llm-providers.md)。

??? question "ローカル LLM は Ollama 固定？"
    いいえ。`local` プロバイダは **OpenAI 互換エンドポイント**（`/v1/chat/completions`）を叩くので、
    Ollama・LM Studio・vLLM・llama.cpp など互換サーバなら何でも使えます。

??? question "Docker の ai-pm で claude-code を使いたい"
    `claude-code`/`codex`/`antigravity` は CLI を subprocess 起動するため、コンテナ内に CLI が必要です。
    現実的にはコンテナ内は `local`（`host.docker.internal`）か `mock`、または **ai-pm をホスト直起動**にします。

## コスト

??? question "全部無料で運用できますか？"
    はい。GitHub private リポ（人数無制限・無料）、既存マシンでのセルフホスト、Tailscale 無料プラン、
    `pg_dump` バックアップ —— いずれも追加費用ゼロで構成できます（[戦略](../strategy/version-control.md)）。
