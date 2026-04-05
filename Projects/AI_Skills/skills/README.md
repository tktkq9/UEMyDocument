# Skills 一覧

このフォルダはSkillsのドキュメント管理用です。
実際に動作するファイルは `C:\Users\galga\.claude\skills\` に配置しています。

| Skill名 | 用途 | トリガー例 |
|---------|------|-----------|
| `research-note` | 調査結果をテンプレートに沿って保存 | "save research", "調査メモを保存" |
| `skill-entry` | 新しいSkillを作成・記録 | "create a skill", "Skillを作って" |
| `claude-md-review` | CLAUDE.mdのレビューと改善 | "CLAUDE.mdを見直して" |
| `save-and-commit` | ファイル保存 + gitコミット | "保存してコミット" |
| `ue5-analyze` | UE5ソースコード解析メモ作成 | "このUE5クラスを解析して" |
| `compact-smart` | コンテキストのスマート圧縮 | "コンテキストを整理", "compact" |

## 新しいSkillを追加するとき
`skill-entry` Skillを使うか、以下の手順で手動追加：
1. このフォルダにドキュメントを作成
2. `C:\Users\galga\.claude\skills\` に実体を配置
3. `description` に `USE WHEN` トリガーを必ず含める
