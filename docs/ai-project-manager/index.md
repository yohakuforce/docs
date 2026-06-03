# AI-Project-Manager（AIマネージャー）概要

!!! abstract "このページの要約"
    **AI-Project-Manager（AI-PM）** は、余白フォース / yohakuforce スイートの **進行管理層** です。[Context-Hub](../context-hub/index.md) が集約した会議メモや課題などの「文脈」を読み取り、受託・社内プロジェクトの進行を **7 つの能力（Plan / Assign / Track / Alert / Overview / Standup / WrapUp）** で日次に回します。タスク・日報・アラートといった進行データは AI-PM 自身の **PostgreSQL** に保持し、API はポート **`:8001`**、設定 GUI は `/settings`（localhost 専用）で提供します。**社内運用アプリ（private リポジトリ）** であり、顧客の生データを外部 LLM / SaaS に転記しないデータ境界を守ります。

---

## 何を解決するか

受託・社内プロジェクトの進行管理では、次のような「人がやると抜け漏れる・続かない」作業が発生します。

- 会議メモや Backlog / Redmine の課題から **タスクを起こす**
- メンバーの状況を見て **担当を割り当てる**
- 毎日 **日報を集め、ブロッカーを拾い、未提出者に催促する**
- タスク遅延・過負荷・未回答を **検知して通知する**
- 経営層・リーダー向けに **日次サマリと総括をまとめる**

AI-PM は、これらを **時刻駆動の自動スケジューラ** と **手動 API / GUI 操作** の両方で回します。AI は提案（DRAFT）を作り、最終判断はリーダーが行う「**AI は提案・人が承認**」を一貫した設計にしています。

> AI に任せられる進行管理は任せ、人は本質的な意思決定（割当の承認・総括の可否）に集中する、という分担を実装したものです。

---

## 7 つの能力（要約）

| 能力 | 内容 |
|---|---|
| **Plan（計画）** | 会議メモや Backlog / Redmine の課題からタスクを自動生成し、プロジェクトに取り込む |
| **Assign（割当）** | メンバーの状況をふまえた担当案（DRAFT）を作成。リーダーが確認・承認 |
| **Track（進捗追跡）** | 日報テンプレを生成・配信し、回答を集約・分析。ブロッカーを抽出。未提出者へ催促 |
| **Alert（アラート）** | タスク遅延・メンバー過負荷・日報未回答を検知して通知 |
| **Overview（経営サマリ）** | タスク状態・日報・アラートを集約した日次サマリとフェーズ進捗を生成 |
| **Standup（スタンドアップ）** | 前日の日報・出来事・アサインをレビューし、問題には DRAFT 入替案を添えてリーダーへ共有 |
| **WrapUp / Status（総括・確認ゲート）** | 当日総括とリーダー確認ゲート、確認後の全体ステータス分析＋未割当 DRAFT アサイン |

各能力の詳細・既定実行時刻・固定実行順は [7 つの能力](capabilities.md) を参照してください。

---

## アーキテクチャと Context-Hub 依存

- 進行データ（**タスク / メンバー / 日報 / アラート / リーダー確認ゲート**）は、AI-PM 自前の **PostgreSQL** に保持します（`USE_DATABASE=true` で永続化）。
- 文脈（**会議メモ・課題**）は、別アプリ **Context-Hub の REST API**（camelCase 契約）から取得します。Context-Hub が無い・繋がない場合は `CONTEXT_HUB_USE_MOCK=true` でモックのまま動作します。
- API は `:8001`、設定 GUI `/settings`・登録 GUI `/register`・運用ガイド `/guide` は **localhost 専用**で提供します。

```text
Context-Hub (:8000, REST)  ──→  AI-Project-Manager (:8001)  ──→  Slack / Sheets / ローカル
        文脈・タスク                  7 能力で進行管理                    配信
```

!!! note "Context-Hub との連携の肝"
    AI-PM の `CONTEXT_HUB_API_KEY` は、Context-Hub 側の `DEV_API_KEY` と **同じ値** にする必要があります。詳細は [デプロイ](deploy.md) と [Context-Hub のドキュメント](../context-hub/index.md) を参照してください。

---

## データ境界（生データを外部に出さない）

AI-PM は **社内運用アプリ（private）** であり、データの取り扱いに明確な境界を設けています。

- 顧客機密は **Context-Hub の取込先（社内 PC）** に閉じます。
- 会議メモからの **タスク抽出は Context-Hub 側（社内 PC・on-prem LLM）で取込時に 1 回だけ** 行われ、生トランスクリプトは外部 API に出ません。
- AI-PM が REST 越しに受け取るのは、**抽象化済みの構造化データのみ** です。
- LLM プロバイダも既定で **課金 API を使わず**、サブスク CLI（Claude Code / Codex / Antigravity）またはローカル LLM（local / ollama）を使います。詳細は [LLM プロバイダ](llm-providers.md) を参照してください。

---

## 配信チャネル

日報・通知の配信は **Slack / Google Sheets / ローカルファイル** のマルチチャネルに対応します。未設定時は安全側に `local_file`（`./.ai-pm/notifications`）→ `in_memory` へフォールバックするため、**トークンが無くても動作確認できます**。

---

## 次に読む

- [デプロイ](deploy.md) — `git clone` → `.env` → `docker compose up` の手順
- [7 つの能力](capabilities.md) — 各能力と日次スケジューラの固定実行順
- [設定 GUI](gui.md) — `/settings`・`/register`・`/guide` の使い方
- [LLM プロバイダ](llm-providers.md) — プロバイダの選び方と Docker での運用注意
