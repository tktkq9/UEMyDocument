# PCG（Procedural Content Generation）概要

- 上位: [[01_worldbuilding_overview]]
- 関連: [[WorldPartition/01_overview]]
- ソース: `Engine/Plugins/PCG/Source/PCG/`（400 h / 547 cpp）

---

## PCG とは

**ノードベースのグラフエディタ** でプロシージャルにコンテンツ（植生・建物・道・地形装飾等）を生成するフレームワーク。ポイントクラウドを操作して最終的にスタティックメッシュやアクタをスポーンする。

---

## アーキテクチャ

```mermaid
graph TD
    Comp[UPCGComponent] -->|executes| Graph[UPCGGraph]
    Graph -->|contains| Node[UPCGNode]
    Node -->|configured by| Settings[UPCGSettings]
    Node -->|runs| Element[IPCGElement]
    Element -->|operates on| Data[UPCGData]
    Data <|-- PointData[UPCGPointData]
    Data <|-- SpatialData[UPCGSpatialData]
    Data <|-- VolumeData[UPCGVolumeData]
    Element -->|uses| Context[FPCGContext]
    Comp -->|registered to| Sub[UPCGSubsystem]
```

---

## 主要クラス

| クラス | 役割 | BP公開 |
|-------|------|--------|
| `UPCGComponent` | アクタに付与する PCG コンポーネント。Generate/Cleanup | Yes |
| `UPCGGraph` | ノードグラフ。評価順序・入出力管理 | Yes |
| `UPCGNode` | グラフ内のノード単位 | No |
| `UPCGSettings` | ノードの設定基底（各ノードタイプが派生） | No |
| `IPCGElement` | ノード実行インターフェース。Execute() を実装 | No |
| `FPCGContext` | 実行コンテキスト。入出力データ・乱数ソース | No |
| `UPCGData` | データ基底クラス | No |
| `UPCGPointData` | ポイントクラウドデータ | No |
| `UPCGSpatialData` | 空間データ基底（Volume/Surface/Spline 等） | No |
| `FPCGPoint` | 個々のポイント（位置・回転・スケール・メタデータ） | No |
| `UPCGSubsystem` | PCG サブシステム（コンポーネント登録・グラフキャッシュ） | Yes |

### 標準ノード（Elements/ — 202 ヘッダ）

| カテゴリ | 代表ノード | 説明 |
|---------|----------|------|
| Sampler | `PointSampler` / `SurfaceSampler` / `MeshSampler` | ジオメトリからポイント生成 |
| Filter | `FilterByAttribute` / `FilterByTag` / `Density` | ポイントのフィルタリング |
| Transform | `Transform` / `ProjectOnSurface` / `BoundsModifier` | ポイントの変形 |
| Spawner | `StaticMeshSpawner` / `SpawnActor` | 最終出力（メッシュ/アクタ生成） |
| Data | `GetActorData` / `GetLandscapeData` / `GetSplineData` | 入力データ取得 |
| Math | `AttributeOperation` / `AttributeFilter` / `Clamp` | 属性演算 |

---

## Details

| ドキュメント | 内容 |
|------------|------|
| [[Details/a_pcg_graph]] | PCG グラフ構造・コンパイル・評価順序 |
| [[Details/b_pcg_elements]] | 標準ノード分類・Sampler/Filter/Transform/Spawner |
| [[Details/c_pcg_data]] | FPCGPoint・PointData・SpatialData・Metadata |
| [[Details/d_custom_nodes]] | カスタムノード実装（C++/BP） |

---

## コード実行フロー

### エントリポイント

```
[BP / C++ 起点]
UPCGComponent::Generate()                                  [PCGComponent.cpp:571]
  └─ UPCGComponent::GenerateLocal(bForce=false)            [PCGComponent.cpp:586]
       └─ GenerateLocalGetTaskId(bForce)                   [PCGComponent.cpp:596]
            └─ UPCGComponent::GenerateInternal()           [PCGComponent.cpp:611]
                 ├─ IsGenerating() / ShouldGenerate() チェック
                 ├─ UPCGSubsystem::ScheduleComponent()     ← タスクID取得
                 │    └─ FPCGGraphCompiler::Compile()      ← グラフをタスクDAG化
                 │         └─ FPCGGraphExecutor::Schedule()
                 ├─ OnPCGGraphStartGenerating デリゲート発火
                 └─ CurrentGenerationTask 保存

[毎フレーム駆動]
UPCGSubsystem::Tick()                                      [PCGSubsystem.cpp:311]
  └─ FPCGGraphExecutor::Execute()
       └─ for each ready task:
            └─ IPCGElement::Execute(FPCGContext)
                 ├─ Context->InputData から FPCGTaggedData 取得
                 ├─ ノード固有処理（Sampler / Filter / Spawner 等）
                 └─ Context->OutputData に結果を格納
       └─ 全タスク完了時:
            └─ UPCGComponent::PostProcessGraph()
                 ├─ Spawner ノードの結果を ISMComponent に反映
                 ├─ OnPCGGraphGenerated デリゲート発火
                 └─ bGenerated = true
```

### フロー詳細

1. **生成リクエスト** — `Generate()` は `Generate_Implementation` 経由で BP に公開（複製対応）。`GenerateLocal()` はローカルのみ（`PCGComponent.cpp:571–588`、[[Reference/ref_pcg_api]]）。
2. **重複防止** — `IsGenerating()` チェックで同時生成を防止。`ShouldGenerate()` で `GenerationTrigger`（`GenerateOnLoad` / `OnDemand` / `AtRuntime`）の妥当性を判定（`PCGComponent.cpp:613`）。
3. **タスクスケジュール** — `UPCGSubsystem::ScheduleComponent()` がグラフをコンパイルし、`FPCGGraphExecutor` の依存グラフ（DAG）にタスクを登録。タスクIDを返す（`PCGComponent.cpp:623`）。
4. **グラフコンパイル** — `FPCGGraphCompiler` が `UPCGGraph` のノード接続を解析して実行順序を決定。SubGraph はインライン展開、ループは展開ノードを生成（[[Details/a_pcg_graph]]）。
5. **ノード実行** — `UPCGSubsystem::Tick` が毎フレーム Ready 状態のタスクを処理。各 `IPCGElement::Execute(Context)` がワーカースレッドで並列実行可能（[[Details/b_pcg_elements]]、[[Details/d_custom_nodes]]）。
6. **データフロー** — `FPCGContext::InputData` (`FPCGDataCollection`) からピン名で入力を取得し、処理結果を `OutputData` に格納。タグ・ピン情報も伝搬（[[Details/c_pcg_data]]）。
7. **キャッシュ** — `IPCGElement::IsCacheable()` が `true` ならハッシュ一致時に再計算スキップ。同一 Settings / 入力データに対して結果再利用。
8. **生成完了** — `Spawner` ノードが ISMC（InstancedStaticMeshComponent）にメッシュインスタンスを書き込み、`OnPCGGraphGenerated` デリゲートで通知。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `UPCGComponent::Generate` | `PCGComponent.cpp:571` | BP/C++ エントリ（複製あり） |
| `UPCGComponent::GenerateLocal` | `PCGComponent.cpp:586` | ローカル生成 |
| `UPCGComponent::GenerateInternal` | `PCGComponent.cpp:611` | スケジュール本体 |
| `UPCGSubsystem::ScheduleComponent` | `PCGSubsystem.cpp` | タスク登録 |
| `UPCGSubsystem::Tick` | `PCGSubsystem.cpp:311` | タスクディスパッチ |
| `FPCGGraphCompiler::Compile` | `PCGGraphCompiler.cpp` | グラフ→DAG 変換 |
| `FPCGGraphExecutor::Execute` | `PCGGraphExecutor.cpp` | DAG 実行 |
| `IPCGElement::Execute` | 各 Element 実装 | ノード固有処理 |
