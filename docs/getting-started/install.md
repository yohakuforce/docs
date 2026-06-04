# クイックスタート（導入）

3 製品を最短で導入する手順です。**順番は「① context-hub →② ai-project-manager」**、
**core はいつでも**（独立）。各製品の詳細は個別ページにリンクしています。

!!! abstract "この手順で出来ること"
    - core を最新版（0.6.0）へ
    - context-hub を立てて文脈基盤を起動
    - ai-project-manager を context-hub に接続して進行管理を開始

---

## 前提（社内 PC で必要なもの）

| 用途 | 必要なもの |
|---|---|
| core | Node.js 20 以上 |
| context-hub | Python 3.12 以上（`pipx` 推奨） |
| ai-project-manager | Docker Desktop（Windows は WSL2 + Docker Desktop）、Git |

---

## ① core のインストール / アップデート

core は npm のグローバルパッケージ（CLI 名 `yohaku`）。

```bash
# 新規インストール
npm install -g @yohakuforce/core

# 旧版が入っている場合は最新（0.6.0）へ
npm install -g @yohakuforce/core@latest

# 確認 → 0.6.0 になればOK
yohaku --version
```

詳細は [core インストール](../core/install.md) / [CLI リファレンス](../core/cli-reference.md)。

---

## ② context-hub のインストール（最初に立てる）

PyPI 公開済み。`pipx` が最短です。

```powershell
pipx install yohakuforce-context-hub
#   意味（ベクトル）検索も使う場合（BGE-M3, 初回約2.3GB DL）:
#   pipx install "yohakuforce-context-hub[embedding]"

cd <作業用フォルダ>                       # ここに .env と data/ ができる
context-hub init --profile quickstart    # .env生成 + DEV_API_KEY を自動発行（画面表示）
context-hub migrate                      # SQLite スキーマ作成
context-hub serve --host 127.0.0.1 --port 8000
```

!!! warning "起動は必ず `context-hub serve` で"
    生の `uvicorn` 起動だと `.env` が読まれず、設定 GUI が認証エラー（401）になります。

!!! tip "DEV_API_KEY を控える"
    `init` が表示する **Admin GUI key (DEV_API_KEY)** を控えてください。
    ③ で ai-project-manager の `CONTEXT_HUB_API_KEY` に使います（両者一致が必須）。
    後から確認: `Select-String DEV_API_KEY .env`（Windows）/ `grep DEV_API_KEY .env`（Mac）。

設定はブラウザの **設定 GUI** が楽です: `http://127.0.0.1:8000/admin` → DEV_API_KEY を貼る →
Sources タブで「プロジェクト作成」。詳細は [設定GUI](../context-hub/admin-gui.md) /
[プロファイル](../context-hub/profiles.md) / [データ取り込み](../context-hub/ingest.md)。

---

## ③ ai-project-manager のインストール（context-hub の後）

GitHub 公開リポ + Docker。**② の context-hub を起動したまま**進めます。

```powershell
git clone https://github.com/yohakuforce/ai-project-manager.git
cd ai-project-manager
copy .env.example .env        # Mac/Linux は cp .env.example .env
```

`.env`（最低限）:

```env
DB_PASSWORD=<強いパスワード>
LLM_PROVIDER=claude-code      # サブスク範囲・API課金なし
CONTEXT_HUB_BASE_URL=http://host.docker.internal:8000/api/v1
CONTEXT_HUB_API_KEY=<②で控えた DEV_API_KEY と同じ値>
CONTEXT_HUB_USE_MOCK=false
NOTIFICATION_CHANNEL=local_file
```

```powershell
docker compose up --build     # db(5433) / app(8001) / migrate
```

!!! danger "鍵の一致が肝"
    ai-project-manager の `CONTEXT_HUB_API_KEY` と context-hub の `DEV_API_KEY` が**一致**して
    いないと接続できません。ここだけ外さなければ繋がります。

動作確認: `http://localhost:8001/health` / 設定 `:8001/settings` / 登録 `:8001/register` / 運用ガイド `:8001/guide`。
詳細は [デプロイ](../ai-project-manager/deploy.md) / [LLM プロバイダ](../ai-project-manager/llm-providers.md)。

---

## 次のステップ

- チームで共有する設計 → [バージョン管理 / チーム共有](../strategy/version-control.md)
- 段階的にプロジェクトへ展開 → [プロジェクト導入戦略](../strategy/adoption.md)
- つまずいたら → [トラブルシューティング](../reference/troubleshooting.md)
