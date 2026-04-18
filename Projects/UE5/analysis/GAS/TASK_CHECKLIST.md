# GAS（Gameplay Ability System）ドキュメント チェックリスト

## 概要
- [ ] `01_gas_overview.md` … GAS 全体概要・アーキテクチャ（更新予定）
- [ ] `_source_map.md` … ソースマップ

---

## サブフォルダ構成

### AbilitySystem — ASC・GameplayAbility・AbilityTask
- [ ] `AbilitySystem/01_overview.md` … AbilitySystem 概要
- [ ] `AbilitySystem/Details/a_asc_lifecycle.md` … ASC 初期化・GrantAbility・RemoveAbility・レプリケーション
- [ ] `AbilitySystem/Details/b_ability_activation.md` … GA 発動フロー・CanActivate・CommitAbility・EndAbility
- [ ] `AbilitySystem/Details/c_ability_task.md` … UAbilityTask・WaitTargetData・WaitInputPress・非同期タスク
- [ ] `AbilitySystem/Details/d_targeting.md` … TargetData・GameplayAbilityTargetActor・TargetDataFilter
- [ ] `AbilitySystem/Reference/ref_asc.md` … UAbilitySystemComponent API
- [ ] `AbilitySystem/Reference/ref_gameplay_ability.md` … UGameplayAbility API
- [ ] `AbilitySystem/Reference/ref_ability_task.md` … UAbilityTask 派生 API

### GameplayEffect — GE 適用・Modifier・ExecutionCalc
- [ ] `GameplayEffect/01_overview.md` … GameplayEffect 概要
- [ ] `GameplayEffect/Details/a_effect_spec.md` … FGameplayEffectSpec・EffectContext・SourceObject
- [ ] `GameplayEffect/Details/b_modifiers.md` … FGameplayModifierInfo・ModifierOp・AggregatorMod
- [ ] `GameplayEffect/Details/c_execution_calc.md` … UGameplayEffectExecutionCalculation・AttributeCapture
- [ ] `GameplayEffect/Details/d_duration_stacking.md` … Duration/Infinite/Instant・Stacking Policy・Period
- [ ] `GameplayEffect/Reference/ref_gameplay_effect.md` … UGameplayEffect API
- [ ] `GameplayEffect/Reference/ref_effect_types.md` … FGameplayEffectSpec / FActiveGameplayEffect API

### GameplayTag — タグ階層・クエリ・マッチング
- [ ] `GameplayTag/01_overview.md` … GameplayTag 概要
- [ ] `GameplayTag/Details/a_tag_system.md` … FGameplayTag 階層構造・FGameplayTagContainer・マッチングロジック
- [ ] `GameplayTag/Details/b_tag_query.md` … FGameplayTagQuery・AnyMatch/AllMatch/NoMatch・ネスト
- [ ] `GameplayTag/Details/c_tag_manager.md` … UGameplayTagsManager・タグ登録・DataTable・Native Tags
- [ ] `GameplayTag/Reference/ref_tag_api.md` … FGameplayTag / FGameplayTagContainer API
- [ ] `GameplayTag/Reference/ref_tag_query_api.md` … FGameplayTagQuery API

### GameplayCue — エフェクト通知・GCManager
- [ ] `GameplayCue/01_overview.md` … GameplayCue 概要
- [ ] `GameplayCue/Details/a_cue_trigger.md` … GameplayCue トリガー・Execute/Add/Remove・GC Tag マッピング
- [ ] `GameplayCue/Details/b_cue_notify.md` … AGameplayCueNotify_Actor / UGameplayCueNotify_Static
- [ ] `GameplayCue/Details/c_cue_manager.md` … UGameplayCueManager・バッチ処理・ネットワーク最適化
- [ ] `GameplayCue/Reference/ref_cue_api.md` … GameplayCue 関連 API

### Attribute — 属性値・計算
- [ ] `Attribute/01_overview.md` … Attribute 概要
- [ ] `Attribute/Details/a_attribute_set.md` … UAttributeSet・PreAttributeChange・PostGameplayEffectExecute
- [ ] `Attribute/Details/b_attribute_data.md` … FGameplayAttributeData・BaseValue/CurrentValue・Clamping
- [ ] `Attribute/Reference/ref_attribute_api.md` … UAttributeSet / FGameplayAttribute API

---

## 進捗サマリ

| サブフォルダ | 概要 | Details | Reference | 完了 |
|------------|------|---------|-----------|------|
| AbilitySystem | 0/1 | 0/4 | 0/3 | 0/8 |
| GameplayEffect | 0/1 | 0/4 | 0/2 | 0/7 |
| GameplayTag | 0/1 | 0/3 | 0/2 | 0/6 |
| GameplayCue | 0/1 | 0/3 | 0/1 | 0/5 |
| Attribute | 0/1 | 0/2 | 0/1 | 0/4 |

**合計**: 概要 1(システム) + ソースマップ 1 + サブフォルダ概要 5 + Details 16 + Reference 9 = **32 ファイル**
