# tech-docs

Claude Code との作業から生まれた技術メモ置き場。

## ルール

- ファイルはルート直下に置く
- 画像は `images/<文書のファイル名から .md を除いたもの>/` に置き、相対パスで参照する
- ファイル名は `YYYY-MM-DD-<slug>.md`（slug は英小文字・数字・ハイフン）
- 先頭に frontmatter を付ける

```markdown
---
title: タイトル
date: YYYY-MM-DD
tags: [tag1, tag2]
---
```

## 探し方

```bash
# タグで探す
grep -l "tags:.*react" *.md
```
