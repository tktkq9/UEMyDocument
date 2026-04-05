# UE5解析プロジェクト

## ソースコードの場所
- UE5クローン: `D:\UnrealEngine\`
- 主要ソース: `D:\UnrealEngine\Engine\Source\`
- 解析メモ保存先: `D:\Learning\Projects\UE5\analysis\`
- コード断片保存先: `D:\Learning\Projects\UE5\snippets\`

## 解析ルール
- C++とBlueprintの両方に言及する
- UE5の公式ドキュメント用語に統一する（Actor, Component, Subsystem等）
- クラス図はMermaid記法で出力
- マクロ（UCLASS, UPROPERTY, UFUNCTION）の意味と効果を注記する

## ソース構成メモ
- `Engine/Source/Runtime/`     … ランタイムモジュール（コアエンジン）
- `Engine/Source/Editor/`      … エディタ専用モジュール
- `Engine/Source/Developer/`   … ビルドツール・開発者向け
- `Engine/Plugins/`            … 公式プラグイン（GAS等）
