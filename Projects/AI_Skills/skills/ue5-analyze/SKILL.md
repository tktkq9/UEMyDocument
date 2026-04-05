---
name: ue5-analyze
description: USE WHEN analyzing, explaining, or documenting Unreal Engine 5 source code, classes, systems, or architecture. Always use this skill when the user asks about UE5 C++ classes, Blueprints, engine subsystems, GameplayAbilitySystem, ActorComponent, Subsystem, or wants analysis saved to file. Trigger phrases: "このUE5クラスを解析", "UE5のコードを読んで", "explain this UE5 class", "UE5のアーキテクチャを教えて", "UE5のソースを調べて". Use even when the user just pastes UE5 C++ code without explicit instructions.
---

UE5のソースコードを解析してメモを作成する。

## 解析ルール
- C++ と Blueprint の両方の観点から説明する
- UE5公式ドキュメントの用語に統一する（`Actor`, `Component`, `Subsystem` 等）
- クラス継承図は Mermaid 記法で出力する
- マクロ（`UCLASS`, `UPROPERTY`, `UFUNCTION`）の意味と効果を必ず注記する

## 出力フォーマット
```markdown
# クラス名 / システム名

## 概要
## 継承階層（Mermaid）
## 主要メソッド・プロパティ
## Blueprintから利用できる機能
## 使用例（C++）
## 関連クラス
```

## 保存先
- 解析メモ: `D:\Learning\Projects\UE5\analysis\`
- コード断片: `D:\Learning\Projects\UE5\snippets\`
