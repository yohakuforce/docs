# 7 つの能力と日次スケジューラ

!!! abstract "このページの要約"
    AI-Project-Manager は **Plan / Assign / Track / Alert / Overview / Standup / WrapUp** の 7 能力で進行管理を回します。内蔵スケジューラ（APScheduler）が **既定で 09:00 朝会 → 14:00 日報 → 17:00 催促 → 17:30 当日総括＋確認ゲート → 30 分間隔のアラートスキャン** を自動実行します。各フェーズの**時刻（時・分）は GUI で調整できますが、実行順序は常に固定**（`standup → report_generate → report_deliver → report_reminder → wrap_up → alert_scan`）です。総括後の **`final_analysis`（全体ステータス分析）だけは時刻駆動ではなく、リーダーが確認ゲートを `proceed` で解決したときに発火**します。

---

## 7 つの能力

### Plan（計画）

会議メモや Backlog / Redmine の課題から **タスクを自動生成** し、プロジェクトに取り込みます。会議メモは Context-Hub に登録した時点で取込時にタスクが抽出され、AI-PM はそれを取り込みます。

```bash
# 会議メモから
curl -X POST http://127.0.0.1:8001/api/v1/plan/extract-from-meeting \
  -H "X-Api-Key: $KEY" -H "Content-Type: application/json" \
  -d '{"project_id":"<PROJECT_UUID>","meeting_id":"<会議ID>"}'

# 課題（Backlog / Redmine）から
curl -X POST http://127.0.0.1:8001/api/v1/plan/import-from-issues \
  -H "X-Api-Key: $KEY" -H "Content-Type: application/json" \
  -d '{"project_id":"<PROJECT_UUID>","source":"backlog","status_filter":"open"}'
```

### Assign（割当）

メンバーの状況をふまえた **担当案（DRAFT）** を作成し、リーダーが承認 / 却下します。**AI は提案・人が承認**が原則です。

```bash
# 割当案を生成
curl -X POST http://127.0.0.1:8001/api/v1/assign/generate-drafts \
  -H "X-Api-Key: $KEY" -H "Content-Type: application/json" \
  -d '{"project_id":"<PROJECT_UUID>"}'

# 承認（却下は /reject）
curl -X POST http://127.0.0.1:8001/api/v1/assign/confirm \
  -H "X-Api-Key: $KEY" -H "Content-Type: application/json" \
  -d '{"project_id":"<PROJECT_UUID>","assignment_id":"<ID>","decided_by":"PM"}'
```

### Track（進捗追跡）

日報テンプレを **生成・配信** し、回答を **集約・分析** します。ブロッカーを抽出し、未提出者へ催促します。

```bash
curl -X POST http://127.0.0.1:8001/api/v1/track/generate-templates -H "X-Api-Key: $KEY" ...
curl -X POST http://127.0.0.1:8001/api/v1/track/deliver           -H "X-Api-Key: $KEY" ...
curl -X POST http://127.0.0.1:8001/api/v1/track/submit-responses  -H "X-Api-Key: $KEY" ...
curl -X POST http://127.0.0.1:8001/api/v1/track/analyze           -H "X-Api-Key: $KEY" ...
```

### Alert（アラート）

**タスク遅延・メンバー過負荷・日報未回答** を検知して通知します。スケジューラの背景ジョブとして 30 分間隔でスキャンします。

### Overview（経営サマリ）

タスク状態・日報・アラートを集約した **日次サマリとフェーズ進捗** を生成します。

### Standup（スタンドアップ）

前日の **日報・出来事（Context-Hub の課題更新）・昨日のアサイン** をレビューし、過負荷・期日超過・ブロック等の問題タスクには別メンバーへの **DRAFT 入替案** を添えてリーダーへ共有します。

### WrapUp / Status（総括・確認ゲート）

当日総括と **リーダー確認ゲート** を起票し、確認後に **全体ステータス分析＋未割当 DRAFT アサイン**（`final_analysis`）を発火します。

---

## 自動スケジューラ（時刻駆動）

内蔵スケジューラ（APScheduler）が設定時刻に日次パイプラインを自動実行します。アプリ起動時に立ち上がり、`SCHEDULER_ENABLED=false` で無効化できます（無効時は手動 API 実行のみ）。

| フェーズ | 既定時刻 | 設定キー（時・分） | 手動エンドポイント |
|---|---|---|---|
| スタンドアップ（朝会） | **09:00** | `STANDUP_HOUR` / `STANDUP_MINUTE` | `POST /api/v1/pipeline/standup` |
| 日報生成 → 配信 | **14:00** | `REPORT_HOUR` / `REPORT_MINUTE` | `POST /api/v1/track/generate-templates` → `/track/deliver` |
| 日報未提出の催促 | **17:00** | `REMINDER_HOUR` / `REMINDER_MINUTE` | （自動。本人 DM ＋リーダー一覧） |
| 当日総括＋確認ゲート | **17:30** | `WRAP_UP_HOUR` / `WRAP_UP_MINUTE` | `POST /api/v1/pipeline/wrap-up` |
| アラートスキャン | **30 分間隔** | `ALERT_SCAN_INTERVAL_MINUTES` | （背景稼働） |
| 全体ステータス分析（`final_analysis`） | **リーダー確認後**（時刻駆動でない） | — | `POST /api/v1/pipeline/final-analysis` |

時刻（時・分）は `/settings` GUI で自由に調整できます。タイムゾーンは `SCHEDULER_TIMEZONE`（既定 `Asia/Tokyo`）で解釈されます。

!!! warning "実行順序は固定（CANONICAL_STEP_ORDER）"
    調整できるのは各フェーズの **時刻（時・分）と間隔だけ** です。実行順序は時刻設定に関わらず常に固定で、次の正準順から入れ替わりません。

    ```text
    standup → report_generate → report_deliver → report_reminder → wrap_up → alert_scan
    ```

    順序の単一の真実（single source of truth）は `src/application/scheduler/schedule_plan.py` の **`CANONICAL_STEP_ORDER`** です。各ジョブ内のステップは必ず `order_steps()` で正準順に整列されます。

---

## 一日の流れ

1. **09:00 スタンドアップ** — 前日の日報・出来事（Context-Hub の課題更新）・昨日のアサインをレビュー。過負荷・期日超過・ブロック等の問題タスクには別メンバーへの **DRAFT 入替案** を作り、スタンドアップとしてリーダーへ共有。
2. **14:00 日報生成 → 配信** — 当日のタスク状態から日報テンプレを生成し配信。
3. **17:00 催促** — 未提出メンバー本人へ DM、リーダーへ未提出者一覧。
4. **17:30 当日総括＋確認ゲート** —
    - 全員提出済みなら総括を生成し、注目タスクを添えてリーダーへ共有。同時に **「タスク状態は最新か」確認ゲート** を起票。
    - 未提出が残るなら **「未提出があるが総括するか」判断ゲート** を起票し、リーダーへ打診。
5. **リーダー確認後（`final_analysis`）** — リーダーが確認ゲートを `proceed` で解決すると、タスク状態・日報提出・アクティブアラートを集約した **全体ステータスレポート** を生成し、未割当タスクへ **DRAFT アサイン** を作成してリーダーへ報告。

---

## リーダー確認ゲート（人の確認で発火）

時刻では進められない「**人の確認・判断**」を待つ仕組みです。`final_analysis` は時刻駆動ではなく、このゲートをリーダーが解決したときに発火します。

ゲートは 2 種類あります。

- **`wrap_up_decision`** — 「未提出があるが総括するか」の判断ゲート。
- **`task_state_current`** — 「タスク状態は最新か」の確認ゲート。

```bash
# 未解決（PENDING）ゲートを一覧
curl -H "X-Api-Key: $KEY" \
  http://127.0.0.1:8001/api/v1/pipeline/<PROJECT_UUID>/gates

# 解決（proceed で後続が発火 / skip で見送り）
curl -X POST \
  http://127.0.0.1:8001/api/v1/pipeline/<PROJECT_UUID>/gates/<GATE_ID>/resolve \
  -H "X-Api-Key: $KEY" -H "Content-Type: application/json" \
  -d '{"decision":"proceed","resolved_by":"leader-1"}'
```

- `wrap_up_decision` を `proceed` → 総括を生成し `task_state_current` ゲートを起票。
- `task_state_current` を `proceed` → `final_analysis`（全体ステータス分析＋未割当 DRAFT アサイン）が発火。

!!! note "ゲートの永続化"
    `USE_DATABASE=true` のとき、ゲートは PostgreSQL（テーブル `leader_gates`、マイグレーション `0002`）に永続化されます。確認が翌日になってもプロセス再起動を跨いで PENDING ゲートが保持され、リーダーはいつでも解決できます。`USE_DATABASE=false`（dev / テスト）ではインメモリ保持です。

---

## バグらないための制御

ユーザー操作で進行が壊れないよう、スケジューラには次の防御が組み込まれています。

- **順序固定**: 調整できるのは時刻だけ。`order_steps()` が常に正準順へ整列。
- **同時刻の安全化**: 複数フェーズが同じ時刻なら 1 ジョブにまとめ、レースを避けて正しい順に逐次実行。
- **多重起動の抑止**: `max_instances=1` ＋ `coalesce=True` で、スキャン間隔を極端に短くしても滞留・多重起動しない。
- **不正値の自動丸め**: 範囲外の時刻 / 分 / 間隔は `clamp_schedule()` が既定値へ丸め、スケジューラは停止しない。
- **障害分離**: 1 プロジェクト / 1 ステップの失敗は記録のみで、他プロジェクトの進行を止めない。
- **起動失敗の分離**: スケジューラの起動失敗は API 本体を巻き込まない（ログのみ・API は継続）。
