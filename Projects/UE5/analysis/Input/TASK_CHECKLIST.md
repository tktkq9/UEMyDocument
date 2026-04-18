# Input ドキュメント チェックリスト

## 概要
- [ ] `01_input_overview.md` … Input 全体概要（更新予定）
- [ ] `_source_map.md` … ソースマップ

---

## サブフォルダ構成

### EnhancedInput — Enhanced Input System コア
- [ ] `EnhancedInput/01_overview.md` … EnhancedInput 概要
- [ ] `EnhancedInput/Details/a_input_action.md` … UInputAction・ValueType（bool/Axis1D/2D/3D）・Accumulation
- [ ] `EnhancedInput/Details/b_mapping_context.md` … UInputMappingContext・優先度・Add/Remove・FEnhancedActionKeyMapping
- [ ] `EnhancedInput/Details/c_input_subsystem.md` … UEnhancedInputLocalPlayerSubsystem・処理パイプライン
- [ ] `EnhancedInput/Reference/ref_enhanced_input_api.md` … UEnhancedInputComponent / UInputAction API

### ModifiersTriggers — 修飾子・トリガー
- [ ] `ModifiersTriggers/01_overview.md` … Modifiers/Triggers 概要
- [ ] `ModifiersTriggers/Details/a_modifiers.md` … UInputModifier_DeadZone/Negate/Scalar/SwizzleAxis/FOVScaling
- [ ] `ModifiersTriggers/Details/b_triggers.md` … UInputTrigger_Down/Pressed/Released/Hold/HoldAndRelease/Tap/Pulse
- [ ] `ModifiersTriggers/Details/c_custom_impl.md` … カスタム Modifier/Trigger 実装パターン・ModifyRaw/UpdateState
- [ ] `ModifiersTriggers/Reference/ref_modifiers_api.md` … UInputModifier_* 全一覧
- [ ] `ModifiersTriggers/Reference/ref_triggers_api.md` … UInputTrigger_* 全一覧

### InputBinding — バインディング・マルチプレイヤー
- [ ] `InputBinding/01_overview.md` … InputBinding 概要
- [ ] `InputBinding/Details/a_bind_action.md` … C++ BindAction・BP バインド・ETriggerEvent 対応
- [ ] `InputBinding/Details/b_player_mappable.md` … UPlayerMappableInputConfig・ランタイムリマッピング
- [ ] `InputBinding/Details/c_input_debug.md` … デバッグビジュアライザ・ShowDebug INPUT
- [ ] `InputBinding/Reference/ref_binding_api.md` … UEnhancedInputComponent バインディング API

---

## 進捗サマリ

| サブフォルダ | 概要 | Details | Reference | 完了 |
|------------|------|---------|-----------|------|
| EnhancedInput | 0/1 | 0/3 | 0/1 | 0/5 |
| ModifiersTriggers | 0/1 | 0/3 | 0/2 | 0/6 |
| InputBinding | 0/1 | 0/3 | 0/1 | 0/5 |

**合計**: 概要 1 + ソースマップ 1 + サブフォルダ概要 3 + Details 9 + Reference 4 = **18 ファイル**
