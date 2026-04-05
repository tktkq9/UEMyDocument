---
name: ue5-analyze
description: USE WHEN analyzing Unreal Engine 5 source code, explaining UE5 classes, or documenting UE5 architecture. Trigger when user asks about UE5 code, classes, or systems.
disable-model-invocation: false
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
`D:\Learning\Projects\UE5\analysis\` にファイルを作成し、コードの断片は `D:\Learning\Projects\UE5\snippets\` に分離して保存する。
