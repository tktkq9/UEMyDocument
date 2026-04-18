# AI ドキュメント チェックリスト

## 概要
- [ ] `01_ai_overview.md` … AI 全体概要（更新予定）
- [ ] `_source_map.md` … ソースマップ

---

## サブフォルダ構成

### BehaviorTree — ビヘイビアツリー
- [ ] `BehaviorTree/01_overview.md` … BehaviorTree 概要
- [ ] `BehaviorTree/Details/a_bt_execution.md` … UBehaviorTreeComponent・ツリー実行フロー・SearchData
- [ ] `BehaviorTree/Details/b_bt_task.md` … UBTTaskNode・ExecuteTask/AbortTask・FinishLatentTask
- [ ] `BehaviorTree/Details/c_bt_decorator.md` … UBTDecorator・CalculateRawCondition・Observer Pattern
- [ ] `BehaviorTree/Details/d_bt_service.md` … UBTService・TickNode・インターバル設定
- [ ] `BehaviorTree/Reference/ref_bt_component.md` … UBehaviorTreeComponent API
- [ ] `BehaviorTree/Reference/ref_bt_nodes.md` … 標準 BTTask/BTDecorator/BTService 一覧

### Blackboard — ブラックボード
- [ ] `Blackboard/01_overview.md` … Blackboard 概要
- [ ] `Blackboard/Details/a_blackboard_data.md` … UBlackboardData・キー型一覧・継承
- [ ] `Blackboard/Details/b_blackboard_component.md` … UBlackboardComponent・SetValue/GetValue・Observer
- [ ] `Blackboard/Details/c_blackboard_sync.md` … ブラックボード同期・複数 AI 共有パターン
- [ ] `Blackboard/Reference/ref_blackboard_api.md` … UBlackboardComponent / UBlackboardData API

### Perception — AI 知覚
- [ ] `Perception/01_overview.md` … Perception 概要
- [ ] `Perception/Details/a_perception_system.md` … UAIPerceptionSystem・Tick・StimuliSource 登録
- [ ] `Perception/Details/b_sense_config.md` … UAISenseConfig_Sight/Hearing/Damage/Touch/Team
- [ ] `Perception/Details/c_perception_component.md` … UAIPerceptionComponent・OnTargetPerceptionUpdated
- [ ] `Perception/Reference/ref_perception_api.md` … UAIPerceptionComponent / UAISense API

### EQS — 環境クエリシステム
- [ ] `EQS/01_overview.md` … EQS 概要
- [ ] `EQS/Details/a_env_query.md` … UEnvQuery・FEnvQueryInstance・非同期実行
- [ ] `EQS/Details/b_generators.md` … UEnvQueryGenerator_SimpleGrid/OnCircle/Actors/PathPoints
- [ ] `EQS/Details/c_tests.md` … UEnvQueryTest_Distance/Trace/Dot/Pathfinding・スコアリング
- [ ] `EQS/Reference/ref_eqs_api.md` … UEnvQueryManager / UEnvQuery API

### Navigation — ナビゲーション
- [ ] `Navigation/01_overview.md` … Navigation 概要
- [ ] `Navigation/Details/a_navmesh.md` … ARecastNavMesh・タイルビルド・Runtime Generation
- [ ] `Navigation/Details/b_pathfinding.md` … FPathFindingQuery・A* 実装・パス平滑化
- [ ] `Navigation/Details/c_nav_agent.md` … UPathFollowingComponent・MoveRequest・AvoidanceManager
- [ ] `Navigation/Details/d_nav_modifiers.md` … NavModifierVolume・NavArea・DynamicObstacle
- [ ] `Navigation/Reference/ref_navigation_api.md` … UNavigationSystemV1 / ARecastNavMesh API
- [ ] `Navigation/Reference/ref_path_following_api.md` … UPathFollowingComponent API

### StateTree — ステートツリー
- [ ] `StateTree/01_overview.md` … StateTree 概要
- [ ] `StateTree/Details/a_state_tree.md` … UStateTree・FStateTreeExecutionContext・State 遷移
- [ ] `StateTree/Details/b_state_tree_tasks.md` … FStateTreeTaskBase・EvaluateConditions・ExternalData
- [ ] `StateTree/Details/c_state_tree_schema.md` … UStateTreeSchema・GameplayTag 条件・Blueprint 連携
- [ ] `StateTree/Reference/ref_state_tree_api.md` … UStateTreeComponent / FStateTreeExecutionContext API

---

## 進捗サマリ

| サブフォルダ | 概要 | Details | Reference | 完了 |
|------------|------|---------|-----------|------|
| BehaviorTree | 0/1 | 0/4 | 0/2 | 0/7 |
| Blackboard | 0/1 | 0/3 | 0/1 | 0/5 |
| Perception | 0/1 | 0/3 | 0/1 | 0/5 |
| EQS | 0/1 | 0/3 | 0/1 | 0/5 |
| Navigation | 0/1 | 0/4 | 0/2 | 0/7 |
| StateTree | 0/1 | 0/3 | 0/1 | 0/5 |

**合計**: 概要 1 + ソースマップ 1 + サブフォルダ概要 6 + Details 20 + Reference 8 = **36 ファイル**
