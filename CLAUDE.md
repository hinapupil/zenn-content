# Zenn Content

Zenn 技術記事の管理リポジトリ。GitHub 連携により、push で自動公開される。

## ディレクトリ構成

```
zenn-content/
├── articles/     # 記事（Markdown）
├── books/        # 本（複数章構成）
└── images/       # 画像（記事スラッグごとにサブディレクトリ）
```

## 記事のフロントマター

```yaml
---
title: "記事タイトル"
emoji: "📝"
type: "tech"                    # tech or idea
topics: ["topic1", "topic2"]    # 最大5つ
published: false                # true で公開
---
```

## 記事の作成

```bash
npx zenn new:article --slug <slug>
```

- スラッグは英語ケバブケース（例: `obsidian-git-sync-guide`）
- スラッグは一度公開したら変更不可

## ナレッジからの記事化フロー

1. Obsidian Vault（`~/workspace/obsidian-vault/knowledge/`）の `status: mature` ノートを元にする
2. 記事を作成したらナレッジノートの `status` を `published`、`zenn_slug` にスラッグを記入
3. ローカルプレビュー: `npx zenn preview`（http://localhost:8000）

## 画像

- `images/<slug>/` にスクリーンショット等を配置
- 記事内での参照: `/images/<slug>/screenshot.png`

## 注意

- `published: true` の記事を push すると即座に公開される
- 下書き中は `published: false` のまま管理
- main ブランチへの push がトリガー
