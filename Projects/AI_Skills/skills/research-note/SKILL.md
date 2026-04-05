---
name: research-note
description: USE WHEN saving, documenting, or recording any research findings, survey results, investigation summaries, or learning notes to a file. Always use this skill when the user has finished researching a topic and wants to preserve the results, or says things like "保存して", "メモしておいて", "ファイルに書き出して", "まとめを保存", "save this research", or "調査メモを作って". Also apply proactively after completing any research task in D:\Learning, even if the user doesn't explicitly ask to save.
---

調査結果を `D:\Learning\Templates\research_note.md` のテンプレートに沿ってMarkdownファイルに保存する。

## 手順
1. 保存先パスを確認（デフォルト: 該当プロジェクトの `notes/` フォルダ）
2. ファイル名は `NN_topic_name.md`（連番_トピック名）形式
3. 以下のテンプレートで作成
4. 保存後に `git add . && git commit -m "feat: add research note - {topic}"` を実行

## テンプレート
```markdown
# タイトル

## 概要
<!-- 何についての情報か1〜2行で -->

## 情報源
- URL: 
- 取得日: YYYY-MM-DD

## 要点
<!-- 箇条書きで3〜5点 -->

## 実践メモ
<!-- 実際に試した内容・結果 -->

## 関連トピック
<!-- ObsidianのWikiリンク [[]] で繋ぐ -->
```
