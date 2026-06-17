# Zenn Posts

Zenn の記事・本を管理するリポジトリです。

## Setup

```sh
npm install
```

## Commands

```sh
npm run new:article
npm run new:book
npm run preview
npm run list:articles
npm run list:books
```

## 投稿フロー

1. `npm run new:article` で記事を作成する
2. `articles/` 配下の Markdown を編集する
3. 必要に応じて `templates/article.md` の frontmatter を参考に整える
4. `npm run preview` で表示を確認する
5. GitHub に push して Zenn と連携する

Zenn の GitHub 連携は Zenn のダッシュボードからこのリポジトリを指定してください。

## 記事の frontmatter

```yaml
---
title: "記事タイトル"
emoji: "📝"
type: "tech"
topics: ["zenn"]
published: false
---
```

`published: false` のまま push すると下書き扱いです。公開するときは `true` に変更してください。
