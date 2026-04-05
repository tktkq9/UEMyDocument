---
name: save-and-commit
description: USE WHEN finishing a task and wanting to save results to a file then commit to git. Trigger when user says "save and commit", "ファイルに保存してコミット", or after completing a research/analysis task.
---

作業結果をファイルに保存してgitコミットする。

## 手順
1. 保存先パスと内容を確認
2. 適切なファイル名で保存（`NN_topic.md` 形式）
3. コミットメッセージは以下の規則に従う：
   - 新規調査メモ: `feat: add research note - {topic}`
   - Skill追加: `feat: add skill - {skill-name}`
   - メモ更新: `update: {filename} - {変更内容}`
   - 設定変更: `config: {変更内容}`
4. `git add {file} && git commit -m "{message}"` を実行

## 注意
- `D:\Learning` リポジトリ配下のファイルが対象
- 機密情報（APIキー等）が含まれていないか確認してからコミット
