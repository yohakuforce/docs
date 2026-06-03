# yohakuforce Help

余白フォース Suite（core / context-hub / ai-project-manager）の公式ヘルプサイト。
[MkDocs Material](https://squidfunk.github.io/mkdocs-material/) で構築し、GitHub Pages で配信します。

公開URL: https://yohakuforce.github.io/docs/

## ローカルでプレビュー

```bash
pip install -r requirements.txt
mkdocs serve        # http://127.0.0.1:8000
```

## 構成

- `docs/` … ページ本体（Markdown）
- `docs/assets/images/` … スクリーンショット等
- `mkdocs.yml` … サイト設定・ナビゲーション
- `.github/workflows/deploy-docs.yml` … main への push で Pages へ自動デプロイ

## 編集の流れ

1. ブランチを切って `docs/` の Markdown を編集
2. `mkdocs build --strict` でリンク切れ等を確認
3. PR → main マージで自動公開
