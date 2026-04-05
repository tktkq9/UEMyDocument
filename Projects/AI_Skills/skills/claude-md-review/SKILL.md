---
name: claude-md-review
description: USE WHEN reviewing, improving, auditing, or cleaning up a CLAUDE.md file. Always trigger when the user says "CLAUDE.mdを見直して", "CLAUDE.mdを改善して", "review CLAUDE.md", "CLAUDE.mdが長くなってきた", or when instructions in CLAUDE.md seem to be getting ignored by Claude. Also use proactively if a CLAUDE.md file exceeds 200 lines or contains redundant/outdated rules.
---

CLAUDE.md をレビューして改善案を提示する。

## チェック項目
1. **行数** — 200行以内か？超えていたら削ぎ落とす候補を提案
2. **構成** — WHY（目的）/ WHAT（技術スタック・構造）/ HOW（コマンド・検証手順）が揃っているか
3. **重複** — linterや他ツールに任せられる指示（コードスタイル等）が含まれていないか
4. **重要度マーク** — 必ず守ってほしい指示に `IMPORTANT:` または `YOU MUST` が付いているか
5. **サブファイル参照** — 長い説明は `@path/to/file.md` で外出しできるか
6. **Progressive Disclosure** — 常に必要な情報だけが書かれているか
7. **陳腐化** — 現在のプロジェクト状況に合わなくなったルールがないか

## 出力形式
- 現状の問題点（箇条書き）
- 改善後の CLAUDE.md 案
- 変更理由の説明
