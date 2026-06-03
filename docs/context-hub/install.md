# インストール

!!! abstract "要約"
    `pipx install yohakuforce-context-hub` で導入し、**`context-hub init` → `migrate` → `serve`** の 3 ステップで起動します。`init` はプロファイル用の `.env` を生成し、開発プロファイル（`quickstart` / `personal`）では **Admin GUI 用の `DEV_API_KEY` を自動発行して画面に表示**します。サーバは必ず **`context-hub serve`** で起動してください（このコマンドがカレントディレクトリの `.env` を読み込みます）。

---

## 必要環境

- **Python 3.12 以上**
- macOS / Linux / Windows（Windows の注意点は後述）

## 1. インストール

`pipx` でのインストールを推奨します（コマンドを独立した仮想環境に隔離できます）。

```bash
pipx install yohakuforce-context-hub
```

`pip` でも導入できます。

```bash
pip install yohakuforce-context-hub
```

### オプション機能（extras）

用途に応じて追加依存を入れます。`quickstart` だけで試すなら不要です。

```bash
# BGE-M3 ローカル埋め込み（personal / production プロファイル。初回 ~2.3 GB のモデルDL）
pip install 'yohakuforce-context-hub[embedding]'

# PostgreSQL サポート（production プロファイル）
pip install 'yohakuforce-context-hub[postgres]'

# Gmail ライブ取り込み（OAuth2 + Gmail API）
pip install 'yohakuforce-context-hub[gmail]'

# PDF / DOCX アップロードのテキスト抽出（pymupdf + python-docx）
pip install 'yohakuforce-context-hub[documents]'

# まとめて
pip install 'yohakuforce-context-hub[embedding,postgres,gmail,documents]'
```

!!! note "pipx で extras を入れる"
    `pipx` でインストール済みの環境に extras を追加するには `pipx inject yohakuforce-context-hub <extra対象パッケージ>` ではなく、`pipx install 'yohakuforce-context-hub[embedding]'` のように **extras 付きで再インストール**するのが簡単です。

## 2. 初期化（init）

作業ディレクトリで `init` を実行します。プロファイル用の `.env` をカレントディレクトリに生成し、`data/` ディレクトリを作成します。

```bash
context-hub init --profile quickstart
```

- `--profile` / `-p`: `quickstart`（既定）/ `personal` / `production`
- `--force` / `-f`: 既存の `.env` を上書き

生成された `.env` のパーミッションは `600`（所有者のみ読み書き）に設定されます。

!!! tip "init が DEV_API_KEY を自動発行する"
    `quickstart` / `personal` プロファイルでは、`init` が **強力なランダム値の `DEV_API_KEY` を生成して `.env` に書き込み、出力にも表示**します。これが [設定GUI（/admin）](admin-gui.md) と REST のデータ呼び出しを認証するキーです。出力の `Admin GUI key (DEV_API_KEY): ...` 行を控えるか、`grep DEV_API_KEY .env` で確認してください。

    ```text
    Wrote .env from profile 'quickstart' (permissions: 600).
    Ensured data/ directory exists.

    Admin GUI key (DEV_API_KEY): 3f9c...（48桁の16進）
      Open the Admin GUI at http://127.0.0.1:8000/admin and paste this key when prompted.
      It is also stored in your .env file.
    ```

!!! warning "production では DEV_API_KEY は無効"
    `production`（`APP_ENV=production`）では `init` は `DEV_API_KEY` を発行せず、サーバ側でも無視されます。Admin GUI には **ADMIN スコープの consumer キーを別途発行**してください（`SECURITY.md` 参照）。production の `init` は `DATABASE_URL` / `SECRET_KEY` の設定を促します。

## 3. マイグレーション（migrate）

データベースのスキーマを適用します。`DATABASE_URL` を見て SQLite / PostgreSQL の適切なランナーを自動選択します。

```bash
context-hub migrate
```

`--dry-run` で適用内容のみ確認、`--target <rev>` で対象リビジョン指定、`--yes` / `-y` で確認プロンプトをスキップできます（production では確認プロンプトが出ます）。詳細は [CLI リファレンス](cli-reference.md) を参照してください。

## 4. 起動（serve）

!!! warning "必ず context-hub serve で起動する"
    サーバは **必ず `context-hub serve` で起動**してください。`serve` は **カレントディレクトリの `.env` を明示的に読み込む**ため、`init` が書いた `DEV_API_KEY` などの設定が反映されます。`uvicorn` を直接叩くと `.env` が読まれず、Admin GUI の認証が通りません。

```bash
# HTTP REST API サーバ（Admin GUI もこの上で動作）
context-hub serve
```

```bash
# stdio MCP サーバとして起動（Claude Desktop / Claude Code 用）
context-hub serve --mcp-only
```

- `--host`（既定 `127.0.0.1`）/ `--port`（既定 `8000`）
- `--mcp-only`: HTTP を立てず stdio MCP サーバのみ起動
- `--reload`: ホットリロード（開発用。production では警告）

!!! note "`--http-only` は存在しません"
    `serve` の既定動作が HTTP REST API です。MCP サーバにしたいときだけ `--mcp-only` を付けます（`--http-only` というフラグはありません）。

## 5. 動作確認

```bash
# ヘルスチェック（認証不要）
curl http://127.0.0.1:8000/health
# {"status":"ok"}

# MCP プロトコルバージョン確認
curl http://127.0.0.1:8000/mcp/version
# {"mcp_protocol_version":"2024-11-05","server":"context-hub","server_version":"0.3.0"}
```

ブラウザで設定画面を開きます。

```bash
open http://127.0.0.1:8000/admin   # macOS（Linux は xdg-open）
```

`init` で表示された `DEV_API_KEY` を貼り付ければ利用開始です。次は [プロファイル](profiles.md) で構成を選び、[データ取り込み](ingest.md) で各ソースを接続してください。

## Windows での注意

- **`pip install` は動作します**（`sqlite-vec` を含め Windows wheel が提供されています）。
- **stock python.org ビルドは FTS-only に自動デグレード**します。`migrate`・取り込み・キーワード検索は動作し、ベクトル検索のみ無効です（起動時に警告ログ）。
- 完全な意味検索が必要なら、**conda / miniforge の Python**（拡張ロード対応）か **`production` プロファイル（PostgreSQL + pgvector）**を使ってください。
