---
name: compact-smart
description: USE WHEN the context window is getting large, or when switching from research phase to implementation phase. Trigger when user says "コンテキストを整理", "compact", or when context reaches 80% usage.
---

コンテキストをスマートに圧縮して重要情報を保持する。

## タイミングの判断基準
- コンテキスト使用率が80%を超えた → 即座に実行
- 大きな機能・調査が完了した → 実行推奨
- 研究フェーズ → 実装フェーズへ切り替え → 実行推奨
- デバッグ中 → 実行しない（履歴が必要）
- 統合作業の直前 → 実行しない

## 実行方法
```
/compact [フォーカス対象の説明]
```

## フォーカス指定の例
- `/compact Focus on the research findings about Claude Code skills`
- `/compact Focus on the UE5 class analysis results`
- `/compact Keep the file paths and key decisions, discard debug logs`

## 注意
- compactは元に戻せない。重要な情報は先にファイルに保存してからcompactを実行する
- compactの前に `save-and-commit` Skillで成果物をファイル化しておく
