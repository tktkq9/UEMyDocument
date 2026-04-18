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
