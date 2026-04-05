---
name: compact-smart
description: USE WHEN the context window is getting large, the conversation is becoming slow, or when switching phases of work. Always trigger when user says "コンテキストを整理", "compact", "context が重い", "会話が長くなってきた", or when context usage exceeds 70-80%. Also recommend proactively when a major research or implementation phase completes and a new phase begins. Do NOT use during active debugging or just before integration work.
---

コンテキストをスマートに圧縮して重要情報を保持する。

## タイミングの判断基準
| 状況 | 対応 |
|------|------|
| コンテキスト使用率 70%超 | 即座に実行推奨 |
| 大きな調査・機能実装が完了 | 実行推奨 |
| 研究フェーズ → 実装フェーズへ切り替え | 実行推奨 |
| デバッグ中 | 実行しない（履歴が必要） |
| 統合作業の直前 | 実行しない |

## 実行前の準備（重要）
compactは元に戻せない。先に `save-and-commit` Skillで成果物をファイルに保存してからcompactを実行する。

## 実行方法
```
/compact [フォーカス対象の説明]
```

## フォーカス指定の例
- `/compact Focus on the research findings about Claude Code skills`
- `/compact Focus on the UE5 class analysis results`
- `/compact Keep the file paths and key decisions, discard debug logs`

フォーカスを指定することで要約の精度が上がり、重要情報が失われにくくなる。
