---
name: skill-entry
description: USE WHEN manually creating a simple Claude Code skill file or documenting an existing skill without the full eval loop. Trigger when user says "Skillを作って", "add a skill", "make a skill for X", or wants a quick skill without benchmarking. For skills that need testing and description optimization, prefer the skill-creator skill instead.
---

新しいSkillをディレクトリ形式で作成し、ドキュメントとして記録する。

## 作成手順
1. `C:\Users\galga\.claude\skills\{skill-name}\SKILL.md` を作成
2. `D:\Learning\Projects\AI_Skills\skills\` にも同じファイルをコピー（記録用）
3. `git add . && git commit -m "feat: add skill - {skill-name}"` を実行

## SKILL.md のフォーマット
```markdown
---
name: skill-name（ハイフン区切り小文字・64文字以内）
description: USE WHEN [具体的なトリガー条件]（1024文字以内）
---

[Skillの内容・手順]
```

## descriptionのベストプラクティス
- `USE WHEN` で始める（発動率が20% → 50%以上に向上）
- トリガーフレーズを複数列挙する（日本語・英語両方）
- 「このSkillを使うべき状況」を具体的に書く
- 積極的にトリガーされるよう、やや"pushy"な表現にする

## 注意
- より高品質なSkillを作りたい場合は `skill-creator` Skillを使う
- 副作用のあるワークフローは `disable-model-invocation: true` を追加してマニュアル起動専用にする
