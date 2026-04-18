# Animation システム全体概要

- 取得対象: `Engine/Source/Runtime/Engine/Classes/Animation/`, `Engine/Plugins/Animation/`
- 上位: [[UE5 解析インデックス]]

---

## UE5 アニメーションシステムの構成

| レイヤー | 主要クラス | 説明 |
|--------|---------|------|
| ランタイム評価 | `UAnimInstance` | AnimBP の実行・ステート管理 |
| グラフ評価 | `FAnimNode_*` | AnimGraph の各ノード |
| Montage | `UAnimMontage` | 再生制御・スロット・ブレンド |
| BlendSpace | `UBlendSpace` | 多次元ブレンド |
| State Machine | `FAnimNode_StateMachine` | ステートマシン管理 |
| IK | `FAnimNode_FABRIK` / Full Body IK | 逆運動学 |
| Control Rig | `UControlRig` | プロシージャルアニメ・リグ制御 |
| Retargeting | `UIKRetargeter` | 骨格間リターゲット |

---

## フレームの流れ

```
USkeletalMeshComponent::TickComponent()
  └─ TickAnimation()
      ├─ UAnimInstance::UpdateAnimation()   // ステート更新・変数計算
      └─ UAnimInstance::ParallelUpdateAnimation()
          └─ FAnimInstanceProxy::UpdateAnimationNode()  // AnimGraph 並列評価
              └─ EvaluateAnimation()
                  └─ FPoseContext → BonePose 計算
                      └─ アニメーション合成・IK 適用
```

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `AnimInstance.h/.cpp` | AnimBP の C++ 基底 |
| `AnimSequence.h/.cpp` | アニメーションクリップ |
| `AnimMontage.h/.cpp` | Montage 再生制御 |
| `BlendSpace.h/.cpp` | BlendSpace 評価 |
| `AnimNode_StateMachine.h/.cpp` | ステートマシン |
| `AnimNode_FullbodyIK.h/.cpp` | Full Body IK ソルバー |
| `ControlRig.h/.cpp` | Control Rig ランタイム |
| `IKRetargeter.h/.cpp` | IK リターゲット |

---

## サブシステムドキュメント

| ドキュメント | 内容 |
|------------|------|
| `Details/a_anim_instance.md` | AnimInstance 評価フロー・AnimBP との関係 |
| `Details/b_anim_montage.md` | Montage 再生・スロット・BlendOut |
| `Details/c_blend_statemachine.md` | BlendSpace・ステートマシン評価 |
| `Details/d_ik.md` | FABRIK・Full Body IK・AnimNode 実装 |
| `Details/e_control_rig.md` | Control Rig グラフ・RigVM 評価 |
| `Details/f_retargeting.md` | IK リターゲット・骨格マッピング |
