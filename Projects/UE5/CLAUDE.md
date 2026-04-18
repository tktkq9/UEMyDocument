# UE5解析プロジェクト

## ソースコードの場所
- UE5クローン: `D:\UnrealEngine\`
- 主要ソース: `D:\UnrealEngine\Engine\Source\`
- 解析メモ保存先: `D:\Learning\Projects\UE5\analysis\`
- コード断片保存先: `D:\Learning\Projects\UE5\snippets\`

## ナビゲーション
- モジュールインデックス: `analysis\_module_index.md` — 全システムのソースパス一覧
- ソースマップ: `analysis\{SystemName}\_source_map.md` — システム別のファイル→クラス対応表
- 手順書: `analysis\HOWTO_create_system_docs.md` — ドキュメント作成手順

## Skill の使い分け
| Skill | 用途 |
|-------|------|
| `ue5-doc` | バッチ文書作成（概要→Details→Reference→Flow） |
| `ue5-dive` | アドホックなソースコード調査・解説 |

## 解析ルール
- C++とBlueprintの両方に言及する
- UE5の公式ドキュメント用語に統一する（Actor, Component, Subsystem等）
- クラス図はMermaid記法で出力
- マクロ（UCLASS, UPROPERTY, UFUNCTION）の意味と効果を注記する

## ソース構成メモ
- `Engine/Source/Runtime/`     … ランタイムモジュール（コアエンジン）
- `Engine/Source/Editor/`      … エディタ専用モジュール
- `Engine/Source/Developer/`   … ビルドツール・開発者向け
- `Engine/Plugins/`            … 公式プラグイン（GAS, EnhancedInput, Niagara, PCG等）
