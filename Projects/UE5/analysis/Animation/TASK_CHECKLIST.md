# Animation ドキュメント チェックリスト

## 概要
- [ ] `01_animation_overview.md` … Animation 全体概要（更新予定）
- [ ] `_source_map.md` … ソースマップ

---

## サブフォルダ構成

### AnimInstance — AnimBP・ステートマシン・BlendTree
- [ ] `AnimInstance/01_overview.md` … AnimInstance 概要
- [ ] `AnimInstance/Details/a_anim_instance.md` … UAnimInstance・FAnimInstanceProxy・NativeUpdate/BlueprintUpdate
- [ ] `AnimInstance/Details/b_state_machine.md` … FAnimNode_StateMachine・Transition 条件・State 構成
- [ ] `AnimInstance/Details/c_blend_tree.md` … FAnimNode_BlendListByBool/Enum・LayeredBoneBlend・PoseDriver
- [ ] `AnimInstance/Details/d_linked_anim.md` … LinkedAnimGraphs・LinkedAnimLayers・テンプレートABP
- [ ] `AnimInstance/Reference/ref_anim_instance.md` … UAnimInstance API
- [ ] `AnimInstance/Reference/ref_anim_nodes.md` … FAnimNode_* 主要ノード API

### Montage — モンタージュ・通知・スロット
- [ ] `Montage/01_overview.md` … Montage 概要
- [ ] `Montage/Details/a_montage_playback.md` … PlayMontage・FAnimMontageInstance・セクション・ブランチング
- [ ] `Montage/Details/b_anim_notify.md` … UAnimNotify・UAnimNotifyState・NotifyBegin/End/Tick
- [ ] `Montage/Details/c_montage_slot.md` … SlotAnimTrack・SlotGroup・ブレンドイン/アウト
- [ ] `Montage/Reference/ref_montage_api.md` … UAnimMontage / FAnimMontageInstance API
- [ ] `Montage/Reference/ref_notify_api.md` … UAnimNotify / UAnimNotifyState API

### IK — インバースキネマティクス
- [ ] `IK/01_overview.md` … IK 概要
- [ ] `IK/Details/a_ik_rig.md` … UIKRigDefinition・IKGoal・FIKRigSolver チェーン
- [ ] `IK/Details/b_ik_solvers.md` … CCDIK・FABRIK・LimbIK・FullBodyIK アルゴリズム
- [ ] `IK/Details/c_ik_anim_node.md` … FAnimNode_IKRig・ランタイム評価・ボーン制約
- [ ] `IK/Reference/ref_ik_api.md` … UIKRigDefinition / FIKRigSolver API

### ControlRig — プロシージャルアニメーション
- [ ] `ControlRig/01_overview.md` … ControlRig 概要
- [ ] `ControlRig/Details/a_control_rig.md` … UControlRig・FRigUnit・FRigVMExecuteContext
- [ ] `ControlRig/Details/b_rig_hierarchy.md` … FRigHierarchy・FRigBoneElement・FRigControlElement
- [ ] `ControlRig/Details/c_rig_graph.md` … ControlRig グラフ評価・Forward/Backward Solve
- [ ] `ControlRig/Reference/ref_control_rig_api.md` … UControlRig / FRigUnit API
- [ ] `ControlRig/Reference/ref_rig_hierarchy_api.md` … FRigHierarchy API

### BlendSpace — ブレンドスペース
- [ ] `BlendSpace/01_overview.md` … BlendSpace 概要
- [ ] `BlendSpace/Details/a_blend_space.md` … UBlendSpace・UBlendSpace1D・サンプル配置・三角形補間
- [ ] `BlendSpace/Details/b_blend_profile.md` … UBlendProfile・ボーン別ブレンドウェイト
- [ ] `BlendSpace/Reference/ref_blend_space_api.md` … UBlendSpace API

### Retargeting — リターゲティング
- [ ] `Retargeting/01_overview.md` … Retargeting 概要
- [ ] `Retargeting/Details/a_ik_retargeter.md` … UIKRetargeter・チェーンマッピング・ルートモーション
- [ ] `Retargeting/Details/b_retarget_profile.md` … FRetargetProfile・ボーンマッピング・スケール調整
- [ ] `Retargeting/Reference/ref_retarget_api.md` … UIKRetargeter API

---

## 進捗サマリ

| サブフォルダ | 概要 | Details | Reference | 完了 |
|------------|------|---------|-----------|------|
| AnimInstance | 0/1 | 0/4 | 0/2 | 0/7 |
| Montage | 0/1 | 0/3 | 0/2 | 0/6 |
| IK | 0/1 | 0/3 | 0/1 | 0/5 |
| ControlRig | 0/1 | 0/3 | 0/2 | 0/6 |
| BlendSpace | 0/1 | 0/2 | 0/1 | 0/4 |
| Retargeting | 0/1 | 0/2 | 0/1 | 0/4 |

**合計**: 概要 1 + ソースマップ 1 + サブフォルダ概要 6 + Details 17 + Reference 9 = **34 ファイル**
