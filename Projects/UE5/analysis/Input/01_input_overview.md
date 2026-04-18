# Input システム全体概要

- 取得対象: `Engine/Plugins/EnhancedInput/Source/`, `Engine/Source/Runtime/Engine/Classes/GameFramework/`
- 上位: [[UE5 解析インデックス]]

---

## UE5 Input システムの構成

UE5 では **Enhanced Input System** が標準。従来の `InputComponent` を拡張し、  
コンテキスト切り替え・修飾子（Modifier）・トリガー条件を柔軟に設定できる。

| 概念 | クラス | 説明 |
|------|--------|------|
| 入力アクション | `UInputAction` | 1 つの論理的な入力定義（ジャンプ・移動等）|
| マッピングコンテキスト | `UInputMappingContext` | キー→アクションのバインドセット |
| モディファイア | `UInputModifier_*` | 入力値の加工（Dead Zone・Negate 等）|
| トリガー | `UInputTrigger_*` | 発火条件（Pressed・Hold・Tap 等）|
| エンハンスドコンポーネント | `UEnhancedInputComponent` | バインド・コールバック管理 |
| ローカルプレイヤーサブシステム | `UEnhancedInputLocalPlayerSubsystem` | コンテキストの Add/Remove |

---

## 入力処理フロー

```
OS/RHI → FSlateApplication::ProcessKeyDownEvent()
  └─ UPlayerInput::ProcessInputStack()
      └─ UEnhancedInputComponent::ProcessInput()
          ├─ UInputMappingContext でアクションを解決
          ├─ UInputModifier::ModifyRaw() で値を加工
          ├─ UInputTrigger::UpdateState() で発火判定
          └─ FInputActionInstance → Delegate 呼び出し
              → BindAction コールバック実行
```

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `EnhancedInputComponent.h/.cpp` | バインド・コールバック管理 |
| `InputAction.h/.cpp` | アクション定義（型・消費設定）|
| `InputMappingContext.h/.cpp` | キーマッピング定義 |
| `InputModifiers.h/.cpp` | 標準モディファイア |
| `InputTriggers.h/.cpp` | 標準トリガー |
| `EnhancedInputLocalPlayerSubsystem.h/.cpp` | コンテキスト管理 |
| `EnhancedPlayerInput.h/.cpp` | 入力スタック処理 |

---

## サブシステムドキュメント

| ドキュメント | 内容 |
|------------|------|
| `Details/a_enhanced_input.md` | Enhanced Input 全体フロー・コンテキスト管理 |
| `Details/b_modifiers_triggers.md` | Modifier・Trigger の種類と実装 |
| `Details/c_input_binding.md` | C++ / BP でのバインド方法・コールバック |
