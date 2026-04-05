# Claude Code と Cursor ベストプラクティス調査レポート

- 取得日: 2026-04-05

---

## 1. Claude Code Skills — 種類と使い方

### 要点
- **Skillsはファイルシステムベースのモジュール機能**。`~/.claude/skills/`（個人）または `.claude/skills/`（プロジェクト）に `SKILL.md` を置くだけで有効化される
- **構成は YAML frontmatter + Markdown**。必須フィールドは `name`（64文字以内・ハイフン区切り小文字）と `description`（1024文字以内）の2つ
- **descriptionの品質が発動率を左右する**。最適化なし→20%、`USE WHEN` トリガー追記→50%、実例を含む評価フック追加→84% まで向上
- **`disable-model-invocation: true` を使うと副作用のあるワークフローをマニュアル起動専用にできる**
- **コミュニティ Skill は85,000件超（2026年3月時点）**。公式 `frontend-design`（27万+ インストール）や `simplify`（コード品質レビュー）が人気

### 情報源
- https://gist.github.com/mellanon/50816550ecb5f3b239aa77eef7b8ed8d
- https://www.firecrawl.dev/blog/best-claude-code-skills
- https://code.claude.com/docs/en/best-practices

### 実用性評価：高

---

## 2. CLAUDE.md — 効果的な書き方とベストプラクティス

### 要点
- **200〜300行以内に収める**（HumanLayer社内では60行以下推奨）。LLMが安定して従える指示は150〜200件が限界
- **WHY / WHAT / HOW パターンで構成する**：プロジェクトの目的（WHY）・技術スタックと構造（WHAT）・コマンドや検証手順（HOW）
- **Progressive Disclosure を活用する**：`@docs/git-instructions.md` のようにサブファイルへの参照で必要時だけ読み込む
- **コードスタイルの指示はlinterに任せる**。CLAUDE.md が肥大化すると重要な指示が無視される
- **定期メンテナンスが必須**：数週間ごとに Claude に「このCLAUDE.mdを見直して改善案を出して」と依頼。`IMPORTANT:` や `YOU MUST` で重要度を強調すると遵守率が上がる

### 情報源
- https://www.humanlayer.dev/blog/writing-a-good-claude-md
- https://www.builder.io/blog/claude-md-guide
- https://www.maxitect.blog/posts/maximising-claude-code-building-an-effective-claudemd

### 実用性評価：高

---

## 3. Claude Code トークン節約テクニック

### 要点
- **コンテキストウィンドウの80%を超えたら複雑タスクを止める**（80/20ルール）。残り20%は軽量・独立タスク専用に温存
- **`/compact` のタイミングを戦略的に選ぶ**：大きな機能完了直後・研究→実装の切り替え時が最適。対象を絞った指示を渡す例：`/compact Focus on the API changes`
- **Subagentで調査タスクを隔離する**：「3ファイル以上にまたがる調査・依存関係監査・テスト実行」はサブエージェントに委譲
- **モデル選択で60〜70% のコスト削減が可能**：軽量タスクのサブエージェントには `model: haiku` を指定。Opus 使用率を20%以下に
- **`/btw` コマンドでコンテキストを消費しない質問ができる**：回答がオーバーレイ表示のみで会話履歴に入らない

### 情報源
- https://code.claude.com/docs/en/costs
- https://claudefa.st/blog/guide/mechanics/context-management
- https://32blog.com/en/claude-code/claude-code-token-cost-reduction-50-percent
- https://www.mintlify.com/affaan-m/everything-claude-code/guides/token-optimization

### 実用性評価：高

---

## 4. Cursor ベストプラクティス

### 要点
- **`.cursorrules` は廃止済み → `.cursor/rules/*.mdc` 形式に移行**：glob パターンで対象ファイルを絞る。追加後は Cursor を再起動
- **Composer は Plan Mode → 実装の2段階で使う**：複雑なリファクタリングは必ず Plan Mode で依存関係を分析させてから実行
- **エージェント実行前に必ず git commit でセーフポイントを作る**：`git add -A && git commit -m "pre-agent-$(date +%s)"` をエイリアス化
- **モデルを用途別に使い分ける**：日常実装は Composer 1.5、複雑なレガシー移行は Claude Opus 4.6、予算重視は DeepSeek V4 Lite
- **`@Codebase` でプロジェクト全体を参照**しつつ、`Playwright + MCP` 連携でテスト自己修復サイクルを構築（2026年の先進的ワークフロー）

### 情報源
- https://github.com/murataslan1/cursor-ai-tips
- https://github.com/digitalchild/cursor-best-practices
- https://www.builder.io/blog/cursor-tips
- https://cursor.com/blog/composer-2

### 実用性評価：高

---

## 5. プロンプトエンジニアリング（AIコーディングツール全般）

### 要点
- **CRISP フレームワーク**：Context（技術環境）→ Role（役割指定）→ Instructions（何をするか）→ Specifications（形式・制約）→ Polish（出力形式）
- **「探索 → 計画 → 実装」を必ず分ける**：一気にコーディングさせると間違った問題を解く
- **検証基準を最初に与える**：「テストを書いてから実装」「スクリーンショットと比較して差分を直す」のように成功条件を明示
- **プロンプトチェーンで複雑タスクを分解**：「まずインタビューして仕様書を作り、新セッションで実装」というパターンが効果的
- **Claude に `AskUserQuestion` でインタビューさせる**：技術実装・UX・エッジケース・トレードオフを深掘りして考慮漏れを潰す

### 情報源
- https://dev.to/iniyarajan86/prompt-engineering-for-developers-10x-your-ai-coding-in-2026-8d6
- https://www.lakera.ai/blog/prompt-engineering-guide
- https://code.claude.com/docs/en/best-practices

### 実用性評価：高

---

## まとめ：即座に試せるアクション

| 優先度 | アクション |
|--------|----------|
| 最高 | Skills の `description` に `USE WHEN` トリガーを追記して発動率50%以上を目指す |
| 最高 | CLAUDE.md を200行以内に収め、WHY/WHAT/HOW で構成する |
| 高 | サブエージェントで調査タスクを隔離しメインコンテキストを守る |
| 高 | Cursor は `.cursor/rules/*.mdc` に移行し、エージェント前に git commit |
| 高 | 実装の前に必ず「探索→計画」フェーズを挟む |
