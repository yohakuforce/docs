# 設定 GUI（/settings・/register・/guide）

!!! abstract "このページの要約"
    AI-Project-Manager は **ブラウザだけで一通り運用を回せる 3 つの GUI** を備えます。**`/settings`** は全設定項目を編集でき、各項目に「なぜ必要 / 取得手順」のヒントが付き、秘匿値はマスク表示されます。**`/register`** はプロジェクトとメンバーを登録・一覧・削除します。**`/guide`** はゼロから日次運用までの手順をブラウザ内で読めます。いずれも **認証不要（`X-Api-Key` 除外）の localhost 専用ツール** なので、外部公開サーバーでは必ず `127.0.0.1` バインドで起動してください。

---

!!! warning "localhost 専用・認証除外"
    `/settings`・`/register`・`/guide` は **ローカル運用者向けで認証を除外**（`X-Api-Key` 不要）しています。外部に公開すると設定が誰でも書き換え可能になるため、**必ず `uvicorn --host 127.0.0.1`（または Docker で localhost のみ公開）で起動**してください。業務 API（`/api/v1/...`）は別途 `X-Api-Key` 必須です。

---

## 設定 GUI — `/settings`

`http://localhost:8001/settings` で開きます。AI-PM の **全設定項目（`Settings` の全属性）** をフォームから編集でき、保存すると `.env` ファイルに書き込まれます。

![設定GUI](../assets/images/ai-pm-settings.png)

特徴は次の通りです。

- **全項目を網羅**: アプリ基本 / データベース / LLM / Context-Hub / 通知 / スケジュール / セキュリティ などのグループに整理。
- **取得ヒント付き**: 各項目に「**なぜ必要 / どこで取得するか**」の説明を必ず添えています。例: `slack_bot_token` には「api.slack.com/apps → OAuth & Permissions → Bot User OAuth Token（xoxb-…）。chat:write スコープとチャンネル招待が必要」が表示されます。
- **秘匿値のマスク**: `app_secret_key`・`database_url`・`*_api_key`・`slack_bot_token` などの秘匿値は **伏字表示** され、未変更なら現状の値を維持します。
- **接続テスト**: 「**Context-Hub 接続テスト**」「**通知チャンネル テスト**」ボタンで疎通を確認できます。

!!! tip "接続テストの使いどころ"
    - **Context-Hub 接続テスト**: `context_hub_base_url`（末尾 `/api/v1`）と `context_hub_api_key`（＝Context-Hub の `DEV_API_KEY`）が一致しているかを確認します。詳細は [デプロイ](deploy.md) を参照。
    - **通知チャンネル テスト**: 選択中の `notification_channel`（slack / google_sheets / local_file / in_memory）へ健全性チェックを送ります。

---

## 登録 GUI — `/register`

`http://localhost:8001/register` で開きます（`/settings` 右上の「🗂 プロジェクト/メンバー登録」からも入れます）。プロジェクトとメンバーを **作成・一覧・削除** できます。

![登録GUI](../assets/images/ai-pm-register.png)

1. **プロジェクトを登録**: 名前（必須）・顧客・ゴール・Context-Hub プロジェクト ID（連携時のみ）を入力。登録すると一覧に **プロジェクト UUID** が出るので控えます（以降の API の `project_id`）。
2. **メンバーを登録**: `external_id`（必須・重複不可。Slack のユーザー ID ＝ DM 宛先に使う）・名前・役割を入力。

!!! note "永続化の挙動"
    登録内容は同じアプリ（スケジューラ・各 API）からそのまま利用できます。`USE_DATABASE=true` なら再起動後も残ります。`false`（インメモリ）ではプロセス内のみ保持されます。

---

## 運用ガイド GUI — `/guide`

`http://localhost:8001/guide` で開きます。**ゼロ → 日次運用までの手順** をブラウザ内で確認できます（公開ヘルプの「運用ガイド」と同内容）。

![運用ガイドGUI](../assets/images/ai-pm-guide.png)

- 最小構成での起動（外部トークン不要）
- 業務 API の認証（`X-Api-Key` ＝ `app_secret_key`）
- Context-Hub との本接続（`context_hub_api_key` の決め方）
- プロジェクト / メンバー登録、日次運用（自動＋手動）、通知連携

---

## 関連ページ

- [デプロイ](deploy.md) — 起動と Context-Hub 連携の前提
- [7 つの能力](capabilities.md) — スケジューラの固定実行順とゲート
- [LLM プロバイダ](llm-providers.md) — `/settings` で選ぶ LLM の選定基準
