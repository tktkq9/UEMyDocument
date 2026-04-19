# PCG 標準ノード・Sampler/Filter/Transform/Spawner

- 上位: [[PCG/01_overview]]
- ソース: `Engine/Plugins/PCG/Source/PCG/Public/Elements/`
          `Engine/Plugins/PCG/Source/PCG/Public/PCGSettings.h`

---

## 概要

PCG の各ノードは **`UPCGSettings`** の派生クラスで定義される。`EPCGSettingsType` で分類され、`FPCGGraphCompiler` がタスクをスケジュールする際に使われる。標準ノードは `Elements/` フォルダに 200+ 種類ある。

---

## UPCGSettings — ノード設定の基底

```cpp
UCLASS(Abstract, MinimalAPI, BlueprintType)
class UPCGSettings : public UObject
{
    // ノードの実行モード
    UPROPERTY(EditAnywhere, Category = Settings, meta=(EditCondition="!IsPropertyReadOnly()"))
    EPCGSettingsExecutionMode ExecutionMode;  // Enabled / Debug / Isolated / Disabled

    // シード値（再現性確保）
    UPROPERTY(EditAnywhere, Category = Settings)
    int32 Seed;

    // 仮想メソッド（派生クラスで実装）
    virtual EPCGSettingsType GetType() const PURE_VIRTUAL(...);
    virtual TArray<FPCGPinProperties> InputPinProperties() const;
    virtual TArray<FPCGPinProperties> OutputPinProperties() const;
    virtual FPCGElementPtr CreateElement() const PURE_VIRTUAL(...);
};
```

### EPCGSettingsExecutionMode

| モード | 動作 |
|-------|------|
| `Enabled` | 通常実行 |
| `Debug` | 実行＋中間データを可視化 |
| `Isolated` | このノードのみ実行（入力を無視してデフォルト値を使用） |
| `Disabled` | スキップ（入力をそのまま出力にパススルー） |

---

## Sampler カテゴリ

ポイントクラウドを生成するノード群。入力として空間データ（地形・スプライン・ボリューム等）を受け取り、ポイントデータを出力する。

### UPCGSurfaceSamplerSettings — サーフェスサンプラー

地形・メッシュ表面上にポイントを配置する。

```cpp
// 主要プロパティ
float PointsPerSquaredMeter;   // 1m² あたりのポイント数
FVector PointExtents;          // 各ポイントのバウンド（半径）
float Looseness;               // セル境界の緩み（重なり許容）
bool bApplyDensityToPoints;    // 密度マップをポイント密度に適用
float PointSteepness;          // 傾斜フィルタ（急斜面を除外）

// 入力ピン
const FName SurfaceLabel = TEXT("Surface");       // サーフェスデータ入力
const FName BoundingShapeLabel = TEXT("Bounding Shape"); // 範囲制限
```

### UPCGSplineSamplerSettings — スプラインサンプラー

スプライン上にポイントを等間隔配置する。道・フェンス・川のポイント生成に使用。

### UPCGVolumeSamplerSettings — ボリュームサンプラー

ボックスボリューム内をランダムサンプリング。

### UPCGTextureSamplerSettings — テクスチャサンプラー

テクスチャのカラー値をポイント属性としてサンプリング（密度マップ作成等）。

---

## Filter カテゴリ

ポイントのフィルタリング・選別を行うノード群。

### UPCGAttributeFilterSettings — 属性フィルタ

属性値の条件でポイントを取り除く。

```cpp
// 例: Density < 0.5 のポイントを除外
// AttributeName: "Density", Operator: LessThan, Value: 0.5
```

### UPCGSelectPointsSettings — ポイント選択

ランダムまたは条件ベースでポイントのサブセットを選択。

### PCGBooleanSelect — 条件分岐

入力ピンに接続されたブール値で 2 つのデータセットを切り替え。

---

## Transform カテゴリ

ポイントの変形・投影を行うノード群。

### UPCGBoundsModifierSettings — バウンド修正

ポイントのスケール・バウンドを修正。StaticMeshSpawner でのメッシュスケール設定に使用。

```cpp
UENUM()
enum class EPCGBoundsModifierMode : uint8
{
    Set,      // バウンドを指定値に設定
    Scale,    // バウンドをスケール
    Intersect,// 既存バウンドと交差
    Include,  // 既存バウンドを拡張
};
```

### PCG Apply Scale To Bounds — スケールをバウンドに適用

ポイントのスケールを Extents に焼き込む。

---

## Spawner カテゴリ

ポイントからメッシュやアクタを生成するノード群。PCG の最終出力担当。

### UPCGStaticMeshSpawnerSettings — スタティックメッシュスポウナー

最も基本的な Spawner。ISM（Instanced Static Mesh）でポイントにメッシュを配置。

```cpp
UCLASS(MinimalAPI, BlueprintType)
class UPCGStaticMeshSpawnerSettings : public UPCGSettings
{
    // メッシュ選択ロジック（ランダム・属性ベース等）
    UPROPERTY(EditAnywhere, Category=Settings, Instanced)
    TObjectPtr<UPCGMeshSelectorBase> MeshSelectorParameters;

    // ISM コンポーネントのパッキング方式
    UPROPERTY(EditAnywhere, Category=Settings, Instanced)
    TObjectPtr<UPCGInstanceDataPackerBase> InstanceDataPackerParameters;

    // コリジョンプロファイル
    UPROPERTY(EditAnywhere, Category=Settings)
    FCollisionProfileName CollisionProfile;

    // GPU で実行するか（デフォルト false）
    virtual bool DisplayExecuteOnGPUSetting() const override { return true; }
};
```

**UPCGMeshSelectorBase の派生クラス**：

| クラス | 説明 |
|-------|------|
| `UPCGMeshSelectorWeighted` | ウェイトでランダム選択 |
| `UPCGMeshSelectorWeightedByCategory` | カテゴリ属性でメッシュを分類して選択 |
| `UPCGMeshSelectorByAttribute` | 属性値で直接メッシュを指定 |

---

## Data カテゴリ

入力データをグラフに取り込むノード群。

| ノード | 説明 |
|-------|------|
| `GetActorData` | ワールドのアクタからデータ取得 |
| `GetLandscapeData` | ランドスケープの高さ・法線・テクスチャ |
| `GetSplineData` | スプラインコンポーネントのデータ |

---

## EPCGSettingsType — ノードの分類

```cpp
UENUM()
enum class EPCGSettingsType : uint8
{
    InputOutput,       // 入出力ノード
    Spatial,           // 空間データ操作
    Density,           // 密度操作
    Blueprint,         // BP カスタムノード
    Metadata,          // メタデータ操作
    Filter,            // フィルタリング
    Sampler,           // ポイント生成
    Spawner,           // メッシュ/アクタ生成
    Subgraph,          // サブグラフ参照
    Debug,             // デバッグ
    Generic,           // 汎用
    Param,             // パラメーター
    HierarchicalGeneration, // 階層的生成
    ControlFlow,       // 制御フロー（Branch / Switch）
    PointOps,          // ポイント操作
    GraphParameters,   // グラフパラメーター
    GPU,               // GPU 実行ノード
    DynamicMesh,       // ダイナミックメッシュ
    DataLayers,        // DataLayer 連携
};
```

---

## コード実行フロー

### エントリポイント

```
[各 Element の共通実行パス]
FPCGGraphExecutor::Execute()
  └─ for each ready FPCGCompiledTask:
       └─ IPCGElement* Element = Task.Settings->CreateElement()
            └─ Element->Execute(FPCGContext*)
                 ├─ PrepareDataInternal(Context)  ← 入力検証
                 ├─ ExecuteInternal(Context)       ← ノード固有処理（下記分岐）
                 └─ PostExecuteInternal(Context)   ← 出力整形

[Sampler 実装例 — UPCGSurfaceSamplerElement::ExecuteInternal]
  ├─ Context->InputData から Surface（UPCGSurfaceData） 取得
  ├─ BoundingShape で範囲クリップ
  ├─ FPCGSurfaceSamplerParams で解像度計算
  │    └─ セルサイズ = 1 / sqrt(PointsPerSquaredMeter)
  ├─ for each セル:
  │    ├─ UPCGSurfaceData::SamplePoint(WorldPos, &Point)
  │    │    └─ 地形ハイトマップから Z 取得・法線計算
  │    ├─ PointSteepness で傾斜フィルタ
  │    └─ FPCGPoint を PointData に追加
  └─ Context->OutputData に UPCGPointData を格納

[Filter 実装例 — UPCGAttributeFilterElement::ExecuteInternal]
  ├─ 入力 PointData を取得
  ├─ UPCGMetadataAttribute を属性名で解決
  └─ for each Point:
       ├─ Attribute->GetValue(Point.MetadataEntry) 取得
       ├─ Operator 評価（LessThan/Equal/Contains 等）
       └─ 条件を満たすポイントのみ出力に追加

[Spawner 実装例 — UPCGStaticMeshSpawnerElement::ExecuteInternal]
  ├─ MeshSelectorParameters->SelectInstances(Context, Points)
  │    ├─ UPCGMeshSelectorWeighted → Seed ベースで重み付き選択
  │    └─ per-instance 属性を FPCGMeshInstanceList に格納
  ├─ InstanceDataPackerParameters->PackInstances(Context)
  │    └─ CustomData float 配列にパッキング
  └─ PostExecute フェーズで:
       └─ UPCGComponent::PostProcessGraph() が
            UInstancedStaticMeshComponent を生成/更新し
            AddInstance(Transform, CustomData) で配置

[Data 入力 — UPCGLandscapeDataGetterElement]
  ├─ Context->SourceComponent->GetWorld() からランドスケープ検索
  ├─ FLandscapeHeightfieldCollisionComponent から高さデータ取得
  └─ UPCGLandscapeData にラップして OutputData へ
```

### フロー詳細

1. **Element 生成** — 各 `UPCGSettings::CreateElement()` がノード固有の `IPCGElement` 実装（`FPCGElement` 派生）を返す。1 回のグラフ実行中は使い回される。
2. **実行モード分岐** — `ExecutionMode=Disabled` は `ExecutePassthroughInternal()` を呼び入力をそのまま出力へ。`Isolated` はデフォルト入力で実行しデバッグしやすくする。
3. **Sampler** — サーフェスサンプラーは地形を `PointsPerSquaredMeter` でセル分割し、各セルに確定的な乱数（Seed 由来）でオフセットしたポイントを配置。`Looseness` でセル境界を跨ぐ重なりを許容し、自然なばらけを作る。
4. **Filter** — `UPCGAttributeFilterElement` は `UPCGMetadataAttribute` 経由で属性アクセス。`FPCGMetadataAttribute<T>` がテンプレートで型安全を保証。条件不一致のポイントは新しい `UPCGPointData` には追加しないだけで、元データは変更しない（イミュータブル）。
5. **Transform** — `UPCGBoundsModifierSettings` の `Mode` により `Set`（置換）/`Scale`（乗算）/`Intersect`（交差）/`Include`（和）を使い分け。ポイントごとの処理はワーカースレッドで並列。
6. **Spawner フロー** — `MeshSelectorBase::SelectInstances()` がポイントごとにメッシュ割当て、`InstanceDataPackerBase::PackInstances()` が ISM の `PerInstanceCustomData` へマテリアル用パラメータをパック。実際の ISMC 生成と `AddInstance()` は `UPCGComponent::PostProcessGraph()` でゲームスレッドで行われる。
7. **データイミュータブル性** — 全ての Element は入力 `UPCGData` を変更せず新しい出力を生成。これによりキャッシュヒット時に安全に結果再利用可能。
8. **GPU 実行** — `DisplayExecuteOnGPUSetting()=true` のノードは `FPCGComputeGraph` 経由で compute shader として実行。大量ポイント処理時のスループット向上に寄与。
9. **EPCGSettingsType** — コンパイラはこの分類でタスクの並列グループを判定。例えば `GPU` タイプは専用キューに投入され、CPU タスクとは独立して実行される。
10. **Debug モード** — `ExecutionMode=Debug` は中間出力を `FPCGContext::DebugData` に保持し、PCG デバッグビューで可視化。デバッグ時のみメモリコスト増。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `IPCGElement::Execute` | `PCGElement.cpp` | 実行エントリ |
| `UPCGSurfaceSamplerElement::ExecuteInternal` | `Elements/PCGSurfaceSampler.cpp` | 地形サンプリング |
| `UPCGAttributeFilterElement::ExecuteInternal` | `Elements/PCGAttributeFilter.cpp` | 属性フィルタ |
| `UPCGStaticMeshSpawnerElement::ExecuteInternal` | `Elements/PCGStaticMeshSpawner.cpp` | メッシュ配置 |
| `UPCGMeshSelectorBase::SelectInstances` | `Elements/PCGMeshSelector.cpp` | メッシュ選択ロジック |
| `UPCGInstanceDataPackerBase::PackInstances` | `Elements/PCGInstanceDataPacker.cpp` | CustomData パッキング |
| `UPCGLandscapeDataGetterElement` | `Elements/PCGLandscapeData.cpp` | 地形データ取得 |
| `UPCGSettings::CreateElement` | `PCGSettings.cpp` | Element ファクトリ |
