# リファレンス：LumenSceneLighting.h / LumenSceneLighting.cpp

- グループ: b - Scene Lighting
- 上位: [[b_lumen_scene_lighting]]
- 関連: [[ref_lumen_scene_data]] | [[ref_lumen_scene_card_capture]] | [[ref_lumen_radiosity]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneLighting.h/cpp`

---

## 概要

Surface Cache のライティング更新パイプライン全体を管理するヘッダ。  
Card ページ単位でのバッチ更新（Direct / Indirect 両方）を担い、  
RDG バッファの割り当て・アトラスへの書き込み・フィードバックループを統括する。

---

## FLumenCardUpdateContext

1 フレームの **Card ページ更新バッチ** を管理する構造体。  
`BuildCardUpdateContext()` で初期化され、Direct Lighting / Radiosity に共用される。

```cpp
class FLumenCardUpdateContext {
public:
    enum EIndirectArgOffset {
        ThreadPerPage = 0 * sizeof(FRHIDispatchIndirectParameters),
        ThreadPerTile = 1 * sizeof(FRHIDispatchIndirectParameters),
        MAX
    };

    FRDGBufferRef CardPageIndexAllocator;
    FRDGBufferRef CardPageIndexData;
    FRDGBufferRef DrawCardPageIndicesIndirectArgs;
    FRDGBufferRef DispatchCardPageIndicesIndirectArgs;

    FIntPoint UpdateAtlasSize;
    uint32 MaxUpdateTiles;
    uint32 UpdateFactor;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `CardPageIndexAllocator` | `FRDGBufferRef` | 更新対象ページ数カウンタ（GPU で Atomic インクリメント）|
| `CardPageIndexData` | `FRDGBufferRef` | 更新対象ページインデックスの配列（GPU で書き込み）|
| `DrawCardPageIndicesIndirectArgs` | `FRDGBufferRef` | DrawIndirect 用引数バッファ（vs Rasterize パス用）|
| `DispatchCardPageIndicesIndirectArgs` | `FRDGBufferRef` | DispatchIndirect 用引数バッファ（2 エントリ: ThreadPerPage / ThreadPerTile）|
| `UpdateAtlasSize` | `FIntPoint` | 今フレームの更新アトラスサイズ（UpdateFactor に応じて縮小）|
| `MaxUpdateTiles` | `uint32` | 最大タイル数（UpdateAtlasSize / TileSize の積）|
| `UpdateFactor` | `uint32` | 更新レート（全ページのうち何分の 1 を今フレームで更新するか）|

### EIndirectArgOffset

| 値 | 意味 |
|----|------|
| `ThreadPerPage` | Dispatch: 1 スレッド = 1 Card ページ |
| `ThreadPerTile` | Dispatch: 1 スレッド = 1 タイル（8×8 px）|

### 使用箇所
- [[ref_lumen_scene_lighting]] `BuildCardUpdateContext()` — 初期化
- [[ref_lumen_scene_direct_lighting]] `RenderDirectLighting()` — Direct Lighting の更新バッチ
- [[ref_lumen_radiosity]] `RenderRadiosity()` — Radiosity の更新バッチ

---

## FLumenCardTileUpdateContext

Card を **8×8 タイル単位** でバッチ処理するコンテキスト。  
Compute Shader の Dispatch 粒度を Card ページ単位からタイル単位に細分化する。

```cpp
struct FLumenCardTileUpdateContext {
    FRDGBufferRef CardTileAllocator;
    FRDGBufferRef CardTiles;
    FRDGBufferRef DispatchCardTilesIndirectArgs;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `CardTileAllocator` | `FRDGBufferRef` | タイル数カウンタ（GPU Atomic）|
| `CardTiles` | `FRDGBufferRef` | タイルインデックス配列（各エントリ: CardPageIndex + TileOffset）|
| `DispatchCardTilesIndirectArgs` | `FRDGBufferRef` | タイル単位 Dispatch 用の Indirect Args（OneThreadPerTile / OneGroupPerTile）|

### 使用箇所
- [[ref_lumen_scene_lighting]] `SpliceCardPagesIntoTiles()` — 生成
- [[ref_lumen_scene_direct_lighting]] `BuildLightTiles()` — タイル単位ライトカリングに使用

---

## FLumenCardScatterParameters

Card のスキャッター描画パラメータ。DrawIndirect / DispatchIndirect 両方に対応する。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenCardScatterParameters, )
    RDG_BUFFER_ACCESS(DrawIndirectArgs, ERHIAccess::IndirectArgs)
    RDG_BUFFER_ACCESS(DispatchIndirectArgs, ERHIAccess::IndirectArgs)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint>, QuadAllocator)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint2>, QuadData)
    SHADER_PARAMETER(uint32, MaxQuadsPerScatterInstance)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `DrawIndirectArgs` | `FRDGBufferRef` | DrawIndirect 引数バッファ |
| `DispatchIndirectArgs` | `FRDGBufferRef` | DispatchIndirect 引数バッファ |
| `QuadAllocator` | `StructuredBuffer<uint>` | 描画クワッド数カウンタ |
| `QuadData` | `StructuredBuffer<uint2>` | クワッドデータ配列 |
| `MaxQuadsPerScatterInstance` | `uint32` | 1 インスタンスあたりの最大クワッド数 |

---

## シェーダークラス

### FRasterizeToCardsVS

Card アトラスへのラスタライズに使うスクリーン空間 VS。  
各 Card ページを適切な UV 範囲にマッピングして描画する。

```cpp
class FRasterizeToCardsVS : public FGlobalShader {
    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
        SHADER_PARAMETER_STRUCT_INCLUDE(FLumenCardScene, LumenCardScene)
        RDG_BUFFER_ACCESS(DrawIndirectArgs, ERHIAccess::IndirectArgs)
        SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint>, CardPageIndexAllocator)
        SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint>, CardPageIndexData)
        SHADER_PARAMETER(FVector2f, IndirectLightingAtlasSize)
    END_SHADER_PARAMETER_STRUCT()
};
```

### 使用箇所
- [[ref_lumen_scene_direct_lighting]] `ClearLumenSceneDirectLighting()` — 描画 VS としてバインド

---

### FClearLumenCardsPS

新規キャプチャ Card のアトラスを消去する PS。  
ターゲット数（1 or 2）をパーミュテーションで切り替える。

```cpp
class FClearLumenCardsPS : public FGlobalShader {
    class FNumTargets : SHADER_PERMUTATION_RANGE_INT("NUM_TARGETS", 1, 2);
    using FPermutationDomain = TShaderPermutationDomain<FNumTargets>;

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
        SHADER_PARAMETER_STRUCT_INCLUDE(FLumenCardScene, LumenCardScene)
    END_SHADER_PARAMETER_STRUCT()
};
```

| パーミュテーション | 値 | 説明 |
|--------------------|-----|------|
| `FNumTargets` | 1 | RT 1 枚（Direct Lighting のみクリア）|
| `FNumTargets` | 2 | RT 2 枚（Direct + Indirect Lighting をクリア）|

### 使用箇所
- [[ref_lumen_scene_direct_lighting]] `ClearLumenSceneDirectLighting()` — PS としてバインド

---

### FCopyCardCaptureLightingToAtlasPS

Card キャプチャで取得した照明を Surface Cache アトラスにコピーする PS。  
Indirect Lighting の有無と Resample の要否をパーミュテーションで制御。

```cpp
class FCopyCardCaptureLightingToAtlasPS : public FGlobalShader {
    class FIndirectLighting : SHADER_PERMUTATION_BOOL("INDIRECT_LIGHTING");
    class FResample        : SHADER_PERMUTATION_BOOL("RESAMPLE");
    using FPermutationDomain = TShaderPermutationDomain<FIndirectLighting, FResample>;

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_RDG_TEXTURE(Texture2D, AlbedoCardCaptureAtlas)
        SHADER_PARAMETER_RDG_TEXTURE(Texture2D, EmissiveCardCaptureAtlas)
        SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DirectLightingCardCaptureAtlas)
        SHADER_PARAMETER_RDG_TEXTURE(Texture2D, RadiosityCardCaptureAtlas)
        SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint>, TileShadowDownsampleFactorAtlas)
    END_SHADER_PARAMETER_STRUCT()
};
```

| パーミュテーション | 説明 |
|--------------------|------|
| `FIndirectLighting = false` | Direct Lighting のみコピー |
| `FIndirectLighting = true` | Direct + Indirect Lighting をコピー |
| `FResample = false` | ゼロ初期化 |
| `FResample = true` | 前フレーム照明をリサンプルしてコピー |

### 使用箇所
- [[ref_lumen_scene_card_capture]] Card キャプチャパイプラインの後半でコピー

---

## ELumenLightType

Lumen の Direct Lighting が扱うライトタイプ列挙。

```cpp
enum class ELumenLightType {
    Directional,  // 平行光源（太陽光・空）
    Point,        // 点光源
    Spot,         // スポットライト
    Rect,         // 矩形ライト（エリアライト）
    MAX
};
```

### 使用箇所
- [[ref_lumen_scene_direct_lighting]] `FLumenGatheredLight::Type` で参照
- [[ref_lumen_scene_direct_lighting]] `FBuildLightTilesCS` のパーミュテーション `FMaxLightSamples` で使用

---

## ELumenDispatchCardTilesIndirectArgsOffset

タイル単位 Dispatch の Indirect Args オフセット列挙。

```cpp
enum class ELumenDispatchCardTilesIndirectArgsOffset {
    OneThreadPerCardTile,  // 1 スレッド = 1 タイル
    OneGroupPerCardTile,   // 1 グループ = 1 タイル
    Num
};
```

---

## FLumenDirectLightingStochasticData

確率的 Direct Lighting の中間データを保持する構造体。

```cpp
struct FLumenDirectLightingStochasticData {
    FRDGBufferRef  CompactedLightSampleData;
    FRDGBufferRef  CompactedLightSampleAllocator;
    FRDGTextureRef SceneDataTexture;
    FRDGTextureRef LightSamples;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `CompactedLightSampleData` | `FRDGBufferRef` | コンパクトされたライトサンプルデータ（ライトインデックス・重みなど）|
| `CompactedLightSampleAllocator` | `FRDGBufferRef` | サンプル数カウンタ（GPU Atomic）|
| `SceneDataTexture` | `FRDGTextureRef` | GBuffer 参照テクスチャ（位置・法線）|
| `LightSamples` | `FRDGTextureRef` | サンプリング結果照明テクスチャ（HW RT へ渡す）|

### 使用箇所
- [[ref_lumen_scene_direct_lighting]] `RenderDirectLightingStochastic()` — 生成・書き込み
- [[ref_lumen_scene_direct_lighting_hwrt]] `TraceLumenHardwareRayTracedDirectLightingShadows()` — Stochastic モード時に HW RT へ渡す

---

## Lumen 名前空間（LumenSceneLighting.h で宣言）

### BuildCardUpdateContext

```cpp
void Lumen::BuildCardUpdateContext(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const TArray<FViewInfo>& Views,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    uint32 UpdateFactor,
    FLumenCardUpdateContext& OutContext);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（PrimitiveGroups 参照）|
| `Views` | `const TArray<FViewInfo>&` | ビュー一覧 |
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | フィードバックバッファ SRV など |
| `UpdateFactor` | `uint32` | 更新レート（大きいほど低頻度）|
| `OutContext` | `FLumenCardUpdateContext&` | 出力コンテキスト |

#### 使用箇所
- [[ref_lumen_scene_lighting]] `FDeferredShadingRenderer::RenderLumenSceneLighting()` — Direct / Radiosity それぞれに対して呼ばれる

#### 内部処理フロー

1. **アトラスサイズと MaxUpdateTiles の決定**
   ```cpp
   // UpdateFactor に応じてアトラスサイズを縮小
   // UpdateAtlasSize = PhysicalAtlasSize / Sqrt(UpdateFactor)
   SetLightingUpdateAtlasSize(PhysicalAtlasSize, UpdateFactor,
       OutContext.UpdateAtlasSize, OutContext.MaxUpdateTiles, OutContext.UpdateFactor);
   ```

2. **バッファのクリア（FClearCardUpdateContextCS）**
   ```cpp
   AddPass(FClearCardUpdateContextCS, PassParams, ...);
   // CardPageIndexAllocator = 0
   // DrawCardPageIndicesIndirectArgs, DispatchCardPageIndicesIndirectArgs = デフォルト値
   ```

3. **優先度ヒストグラムの構築（FBuildPageUpdatePriorityHistogramCS）**
   ```cpp
   // フィードバックバッファが有効なら: FSurfaceCacheFeedback パーミュテーション有効
   // 各 CardPage の更新優先度バケットに票を積む
   AddPass(FBuildPageUpdatePriorityHistogramCS, ...);
   ```

4. **最大更新バケットの選択（FSelectMaxUpdateBucketCS）**
   ```cpp
   // ヒストグラムから MaxUpdateTiles に収まる最大バケットを求める
   AddPass(FSelectMaxUpdateBucketCS, ...);
   ```

5. **更新リストの構築（FBuildCardsUpdateListCS）**
   ```cpp
   // 選択バケット以上の優先度を持つ CardPage を CardPageIndexData に書き込む
   AddPass(FBuildCardsUpdateListCS, ...);
   ```

6. **Indirect Args の設定（FSetCardPageIndexIndirectArgsCS）**
   ```cpp
   // CardPageIndexAllocator の値から DrawIndirectArgs / DispatchIndirectArgs を設定
   AddPass(FSetCardPageIndexIndirectArgsCS, ...);
   ```

---

### SpliceCardPagesIntoTiles

```cpp
void Lumen::SpliceCardPagesIntoTiles(
    FRDGBuilder& GraphBuilder,
    const FLumenCardUpdateContext& CardUpdateContext,
    FLumenCardTileUpdateContext& OutCardTileUpdateContext);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `CardUpdateContext` | `const FLumenCardUpdateContext&` | ページ単位の更新コンテキスト |
| `OutCardTileUpdateContext` | `FLumenCardTileUpdateContext&` | 出力: タイル単位コンテキスト |

#### 使用箇所
- [[ref_lumen_scene_direct_lighting]] `RenderDirectLighting()` 内で呼ばれ、ライトタイルカリングに使用

#### 内部処理フロー

```cpp
// FSpliceCardPagesIntoTilesCS シェーダーを Dispatch
// 各 CardPage を CardTileSize(8) でタイル分割
// CardTiles バッファに (PageIndex, TileOffset) を書き込む
AddPass(FSpliceCardPagesIntoTilesCS, PassParams, ...);
// FInitializeCardTileIndirectArgsCS で DispatchCardTilesIndirectArgs を設定
```

---

### CombineLumenSceneLighting

```cpp
void Lumen::CombineLumenSceneLighting(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FLumenSceneFrameTemporaries& FrameTemporaries);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `View` | `const FViewInfo&` | ビュー情報 |
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | Direct/Indirect/Final アトラス |

#### 使用箇所
- [[ref_lumen_scene_rendering]] `RenderLumenSceneLighting()` の最後に呼ばれる

#### 内部処理フロー

```cpp
// FLumenCardCombineLightingCS を実行
// Albedo × (DirectLighting + DiffuseColorBoost × IndirectLighting) → FinalLightingAtlas
PassParams->AlbedoAtlas           = FrameTemporaries.AlbedoAtlas;
PassParams->DirectLightingAtlas   = FrameTemporaries.DirectLightingAtlas;
PassParams->IndirectLightingAtlas = FrameTemporaries.IndirectLightingAtlas;
PassParams->RWFinalLightingAtlas  = GraphBuilder.CreateUAV(FrameTemporaries.FinalLightingAtlas);
PassParams->DiffuseColorBoost     = CVarLumen... ; // 通常 1.0
AddComputePass(FLumenCardCombineLightingCS, ...);
```

---

> [!note]- SetDirectLightingDeferredLightUniformBuffer — Deferred Light UB をセット
> 
> ```cpp
> void Lumen::SetDirectLightingDeferredLightUniformBuffer(
>     const FViewInfo& View,
>     const FLightSceneInfo* LightSceneInfo,
>     TUniformBufferRef<FDeferredLightUniformStruct>& OutUniformBuffer);
> ```
> 
> **説明**: `FDeferredLightUniformStruct` を `LightSceneInfo` から構築し、ビューローカルな Deferred Light UB を生成する。
> 
> **使用箇所**: [[ref_lumen_scene_direct_lighting]] `GatherLights()` — 各 FLumenGatheredLight の `DeferredLightUniformBuffers` に格納

---

> [!note]- GetDirectLightingAtlasFormat / GetIndirectLightingAtlasFormat — アトラスフォーマット取得
> 
> ```cpp
> EPixelFormat Lumen::GetDirectLightingAtlasFormat();
> EPixelFormat Lumen::GetIndirectLightingAtlasFormat();
> ```
> 
> **戻り値**: `PF_FloatRGBA`（HDR 照明アトラス用）
> 
> **使用箇所**: [[ref_lumen_scene_data]] `FLumenSceneData::AllocateCardAtlases()` — アトラスフォーマット選択

---

> [!note]- LumenSceneLighting::UseAsyncCompute — 非同期計算の可否
> 
> ```cpp
> bool LumenSceneLighting::UseAsyncCompute();
> ```
> 
> **戻り値**: `bool` — 非同期コンピュートキューを使うか
> 
> **内部動作**: `GSupportsEfficientAsyncCompute && r.LumenScene.Lighting.AsyncCompute != 0`
> 
> **使用箇所**: [[ref_lumen_scene_rendering]] `RenderLumenSceneLighting()` — Async Compute フェンス設定

---

## LumenSceneDirectLighting 名前空間

Direct Lighting 計算に関するユーティリティ関数群。

```cpp
namespace LumenSceneDirectLighting {
    constexpr uint32 NumBatchableLightTypes = 3; // Point / Spot / Rect

    float GetMeshSDFShadowRayBias();
    float GetHeightfieldShadowRayBias();
    float GetGlobalSDFShadowRayBias();
    float GetHardwareRayTracingShadowRayBias();

    bool UseStochasticLighting(const FSceneViewFamily& ViewFamily);
    bool UseLightTilesPerLightType();

    void TraceLumenHardwareRayTracedDirectLightingShadows(...);
    FRDGBufferSRVRef TraceLumenHardwareRayTracedDebug(...);
}
```

### 使用箇所まとめ

| 関数 | 使用元 |
|-----|--------|
| `GetMeshSDFShadowRayBias()` | [[ref_lumen_scene_direct_lighting]] `FLumenSceneDirectLightingTraceDistanceFieldShadowsCS` パラメータ設定 |
| `UseStochasticLighting()` | [[ref_lumen_scene_direct_lighting]] `RenderDirectLighting()` — パス分岐 |
| `UseLightTilesPerLightType()` | [[ref_lumen_scene_direct_lighting]] `BuildLightTiles()` — タイル分割戦略 |

---

## 主要 CVar（LumenSceneLighting.cpp）

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.Lighting.ForceLightingUpdate` | 0 | 毎フレーム全 Card を強制更新（デバッグ用）|
| `r.LumenScene.Lighting.Feedback` | 1 | フィードバックバッファの利用有無 |
| `r.LumenScene.Lighting.AsyncCompute` | 0 | 非同期コンピュートの利用有無 |
| `r.LumenScene.DirectLighting.UpdateFactor` | 32 | Direct Lighting の更新レート（1/32 ずつ更新）|
| `r.LumenScene.Radiosity.UpdateFactor` | 64 | Radiosity の更新レート（1/64 ずつ更新）|
| `r.LumenScene.Lighting.LightingStats` | 0 | ライティング統計の画面出力（0=無効、1-3=詳細度）|
| `r.LumenScene.Lighting.DiffuseColorBoost` | 1.0 | Indirect Lighting のカラーブースト係数 |

---

## Card ライティング更新フロー

```
FDeferredShadingRenderer::RenderLumenSceneLighting()
  │
  ├─ LumenRadiosity::InitFrameTemporaries()  ← プローブアトラスの初期化
  │
  ├─ BuildCardUpdateContext(UpdateFactor=32)  ← Direct Lighting 更新対象ページ選定
  ├─ SpliceCardPagesIntoTiles()              ← ページ→タイルに細分化
  │
  ├─ [Direct Lighting]
  │   ├─ ClearLumenSceneDirectLighting()     ← Direct Lighting アトラスをクリア
  │   ├─ GatherLights()                      ← シーン内有効ライトを収集
  │   ├─ BuildLightTiles()                   ← タイルごとのライトカリング
  │   ├─ UseStochasticLighting() ?
  │   │   ├─ true:  RenderDirectLightingStochastic()
  │   │   └─ false: RenderDirectLightingStandard()
  │   └─ CompositeDirectLighting()           → Direct Lighting Atlas に書き込み
  │
  ├─ BuildCardUpdateContext(UpdateFactor=64)  ← Radiosity 更新対象ページ選定
  ├─ [Radiosity / Indirect Lighting]
  │   ├─ LumenRadiosity::AddRadiosityPass()  ← プローブトレース・SH 変換
  │   └─ → Indirect Lighting Atlas に書き込み
  │
  └─ CombineLumenSceneLighting()             ← Final Color Atlas へ合成
```
