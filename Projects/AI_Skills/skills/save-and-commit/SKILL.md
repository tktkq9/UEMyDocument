---
name: save-and-commit
description: USE WHEN saving work results to a file and committing to git. Always trigger when the user says "保存してコミット", "save and commit", "ファイルに書き出してコミット", "コミットして", or after completing any research, analysis, or skill-creation task in D:\Learning that produced content worth preserving. Apply proactively at the end of a work session even if not explicitly requested.
---

作業結果を `D:\Learning` 配下のファイルに保存してgitコミットする。

## 手順
1. 保存先パスと内容を確認
2. 適切なファイル名で保存（`NN_topic.md` 形式）
3. 以下のコミットメッセージ規則に従う
4. `git add {file} && git commit -m "{message}"` を実行

## コミットメッセージ規則
| 種別 | フォーマット |
|------|------------|
| 新規調査メモ | `feat: add research note - {topic}` |
| Skill追加 | `feat: add skill - {skill-name}` |
| メモ更新 | `update: {filename} - {変更内容}` |
| 設定変更 | `config: {変更内容}` |

## 注意
- `D:\Learning` リポジトリ配下のファイルのみ対象
- 機密情報（APIキー等）が含まれていないか確認してからコミット
