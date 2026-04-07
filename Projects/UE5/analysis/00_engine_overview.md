# UE5 ソース構成概要

- 取得日: 2026-04-05
- 対象: D:\UnrealEngine\（公式GitHubクローン）
- 下位: [[01_rendering_overview]]

---

## トップレベル構成

```
D:\UnrealEngine\
├── Engine/
│   ├── Source/
│   │   ├── Runtime/    … ランタイムモジュール（188個）
│   │   ├── Editor/     … エディタ専用モジュール
│   │   └── Developer/  … ビルドツール・開発者向け
│   └── Plugins/        … 公式プラグイン
├── Samples/            … サンプルプロジェクト
└── Templates/          … プロジェクトテンプレート
```

---

## 主要 Runtime モジュール

| モジュール | パス | 役割 |
|-----------|------|------|
| **Core** | `Runtime/Core/` | 基盤型・コンテナ・メモリ管理 |
| **CoreUObject** | `Runtime/CoreUObject/` | UObjectシステム・リフレクション |
| **Engine** | `Runtime/Engine/` | Actor・Component・World・GameMode |
| **Renderer** | `Runtime/Renderer/` | レンダリングパイプライン |
| **PhysicsCore** | `Runtime/PhysicsCore/` | 物理演算基盤 |
| **AIModule** | `Runtime/AIModule/` | AIコントローラ・ナビメッシュ |
| **Slate / SlateCore** | `Runtime/Slate/` | UIフレームワーク基盤 |
| **UMG** | `Runtime/UMG/` | Widget・Blueprint UI |
| **Net** | `Runtime/Net/` | ネットワーク・レプリケーション |
| **GameplayTags** | `Runtime/GameplayTags/` | GameplayTagシステム |

---

## 主要 Plugin（Gameplay系）

| プラグイン | パス | 役割 |
|-----------|------|------|
| **GameplayAbilities (GAS)** | `Plugins/Runtime/GameplayAbilities/` | アビリティシステム全体 |
| **ModularGameplay** | `Plugins/Runtime/ModularGameplay/` | モジュラーゲームプレイ |
| **MassGameplay** | `Plugins/Runtime/MassGameplay/` | ECSベースの大規模AI |
| **GameplayStateTree** | `Plugins/Runtime/GameplayStateTree/` | ステートツリー |

---

## 解析優先順位（予定）

1. `Core` / `CoreUObject` … UObjectの仕組みを理解
2. `Engine` … Actor / Component / World の構造
3. `GameplayAbilities` … GASの全体設計
4. `Renderer` … レンダリングパイプライン
