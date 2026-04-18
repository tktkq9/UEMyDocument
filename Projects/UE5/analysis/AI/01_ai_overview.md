# AI システム全体概要

- 取得対象: `Engine/Source/Runtime/AIModule/`, `Engine/Plugins/AI/`
- 上位: [[UE5 解析インデックス]]

---

## UE5 AI システムの構成

| システム | 主要クラス | 説明 |
|--------|---------|------|
| BehaviorTree | `UBehaviorTree`, `UBTTask_*`, `UBTDecorator_*` | タスクベースの AI 行動定義 |
| Blackboard | `UBlackboardComponent` | BT の共有データストア |
| AIController | `AAIController` | AI ポーンの制御・知覚・移動 |
| Perception | `UAIPerceptionComponent` | 視覚・聴覚・ダメージ感知 |
| EQS | `UEnvQueryManager` | 環境クエリ（最適位置選択）|
| NavMesh | `ARecastNavMesh` | ナビゲーションメッシュ |
| NavAgents | `UNavigationSystemV1` | パスファインド・移動 |
| Mass AI | `UMassEntitySubsystem` | 大規模 AI（群衆シミュレーション）|

---

## 実行フロー（BehaviorTree）

```
AAIController::Tick()
  └─ UBrainComponent::TickComponent()
      └─ UBehaviorTreeComponent::TickComponent()
          └─ FBehaviorTreeInstance::ExecuteCurrentNode()
              ├─ UBTDecorator::CalculateRawConditionValue()  // 条件チェック
              └─ UBTTask::ExecuteTask()                      // タスク実行
                  └─ UAITask_MoveTo / UAITask_*
```

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `AIController.h/.cpp` | AI 制御の基底クラス |
| `BehaviorTreeComponent.h/.cpp` | BT ランタイム |
| `BlackboardComponent.h/.cpp` | Blackboard データ管理 |
| `AIPerceptionComponent.h/.cpp` | 知覚システム |
| `EnvQueryManager.h/.cpp` | EQS クエリ管理 |
| `NavigationSystemV1.h/.cpp` | ナビゲーションシステム |
| `RecastNavMesh.h/.cpp` | Recast NavMesh |
| `MassEntitySubsystem.h/.cpp` | Mass Entity フレームワーク |

---

## サブシステムドキュメント

| ドキュメント | 内容 |
|------------|------|
| `Details/a_behavior_tree.md` | BT ノード種別・実行フロー・サービス |
| `Details/b_blackboard.md` | Blackboard キー・同期・デバッグ |
| `Details/c_ai_perception.md` | 視覚・聴覚・感知コンフィグ |
| `Details/d_eqs.md` | EQS テスト・Generator・コンテキスト |
| `Details/e_navigation.md` | NavMesh ビルド・パスファインド・エージェント |
| `Details/f_mass_ai.md` | Mass Entity・Processor・Fragment |
