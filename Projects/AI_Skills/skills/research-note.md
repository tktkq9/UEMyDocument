---
name: research-note
description: USE WHEN saving research results, investigation summaries, or survey findings to a file. Use this when the user asks to "save", "document", or "record" research about any topic.
---

調査結果をテンプレートに沿ってMarkdownファイルに保存する。

## 手順
1. 保存先パスをユーザーに確認（デフォルト: 該当プロジェクトの `notes/` フォルダ）
2. `D:\Learning\Templates\research_note.md` のテンプレートを使用
3. ファイル名は `NN_topic_name.md`（連番_トピック名）形式
4. 保存後に `git add . && git commit -m "feat: add research note - {topic}"` を実行

## 出力形式
```markdown
# タイトル
- 取得日: YYYY-MM-DD
## 概要
## 情報源
## 要点
## 実践メモ
## 関連トピック
```
