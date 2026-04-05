---
name: skill-entry
description: USE WHEN creating a new Claude Code skill or documenting an existing skill. Trigger when user says "create a skill", "add a skill", or "make a skill for X".
---

新しいSkillを作成し、ドキュメントとして記録する。

## 手順
1. `D:\Learning\Templates\skill_entry.md` のテンプレートを使用してドキュメントを作成
2. `D:\Learning\Projects\AI_Skills\skills\` に保存（記録用）
3. `C:\Users\galga\.claude\skills\` に実際のSkillファイルを作成
4. Skillファイルの `description` には必ず `USE WHEN` トリガーを含める

## Skillファイルのフォーマット
```markdown
---
name: skill-name（ハイフン区切り小文字・64文字以内）
description: USE WHEN [具体的なトリガー条件]（1024文字以内）
---

[Skillの内容・手順]
```

## 重要
- `description` の品質が発動率を左右する（`USE WHEN` なしは発動率20%、ありは50%以上）
- 副作用のあるワークフローは `disable-model-invocation: true` を追加してマニュアル起動専用にする
