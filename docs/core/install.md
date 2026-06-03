# インストール / アップデート

**@yohakuforce/core**（CLI 名 `yohaku`）の導入・更新手順です。npm からのグローバルインストールが基本で、お試しなら `npx` で単発実行もできます。Node.js 20 以上が必須です。

---

## 前提：Node.js 要件

core は **Node.js 20.0.0 以上**を必要とします（`package.json` の `engines.node` に `>=20.0.0` を指定）。

```bash
node --version
# v20.x.x 以上であることを確認
```

!!! warning "Node 20 未満では動きません"
    core は ES Modules / `better-sqlite3`（ネイティブアドオン）に依存します。Node 18 以下では起動に失敗します。`nvs` / `nodenv` / `Volta` などで Node 20+ を有効にしてから導入してください。

---

## グローバルインストール（推奨）

```bash
npm install -g @yohakuforce/core
```

インストール後、`yohaku` コマンドがどこからでも使えるようになります。バージョンを確認しましょう。

```bash
yohaku --version
# 0.6.0
```

ヘルプを表示して全コマンドの一覧を確認できます。

```bash
yohaku --help
```

!!! note "ネイティブビルドについて"
    `better-sqlite3` はインストール時にネイティブビルド（またはプリビルド取得）が走ります。社内プロキシ環境などでビルドが失敗する場合は、ビルドツール（macOS: Xcode CLT、Linux: `build-essential` / `python3`）が入っているか確認してください。

---

## アップデート

最新版へ更新するには、`@latest` を付けて再インストールします。

```bash
npm install -g @yohakuforce/core@latest
yohaku --version
```

特定バージョンに固定したい場合はバージョンを明示します。

```bash
npm install -g @yohakuforce/core@0.6.0
```

!!! tip "アップデート後の再生成"
    レンダリングロジックが更新されることがあるため、アップデート後は `yohaku sync`（または `yohaku graph build` + `yohaku render`）を実行して設計書を再生成しておくと安心です。生成物は決定的なので、内容に差分が出たぶんだけがバージョン間の改善点です。

---

## npx で 1 回だけ試す

グローバルインストールせずに試したい場合は、`npx` で単発実行できます。Salesforce DX プロジェクトのルートで実行してください。

```bash
cd /path/to/your-salesforce-project
npx -p @yohakuforce/core yohaku init --bootstrap --profile minimal
```

`-p @yohakuforce/core` でパッケージを指定し、その中の `yohaku` バイナリを呼び出す形です。バージョン確認も同様にできます。

```bash
npx -p @yohakuforce/core yohaku --version
```

!!! note "npx は CI / お試し向け"
    `npx` は毎回パッケージを解決するため、日常運用では `npm install -g` のほうが高速です。`npx` は CI ワンショットや「まず動かしてみる」用途に向いています。

---

## インストール直後の最初の一歩

導入できたら、Salesforce DX プロジェクトのルートでブートストラップを実行します。`init --bootstrap` は **init + graph build + render** を 1 コマンドでまとめて行います。

```bash
cd /path/to/your-salesforce-project
yohaku init --bootstrap --profile minimal
```

以降は日常運用コマンド `yohaku sync`（graph build --incremental + render を一括）で最新化します。

```bash
yohaku sync
```

各コマンドの詳細は [CLI リファレンス](cli-reference.md) を参照してください。

---

## アンインストール

```bash
npm uninstall -g @yohakuforce/core
```

プロジェクト側に生成された `.yohaku/`（設定・グラフ）や `docs/generated/`（設計書）はそのまま残ります。不要なら手動で削除してください。なお `.yohaku/graph.sqlite` は再生成物なので削除しても `yohaku graph build` で復元できます（[.yohaku 設定](configuration.md) 参照）。
