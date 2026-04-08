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
    // DispatchIndirectArgs のオフセット定数
    enum EIndirectArgOffset {
        ThreadPerPage = 0 * sizeof(FRHIDispatchIndirectParameters), // 1スレッド = 1ページ
        ThreadPerTile = 1 * sizeof(FRHIDispatchIndirectParameters), // 1スレッド = 1タイル
    };

    FRDGBufferRef CardPageIndexAllocator;          // 更新対象ページ数カウンタ
    FRDGBufferRef CardPageIndexData;               // 更新対象ページインデックス配列
    FRDGBufferRef DrawCardPageIndicesIndirectArgs; // 描画用 Indirect Args
    FRDGBufferRef DispatchCardPageIndicesIndirectArgs; // Dispatch 用 Indirect Args（2エントリ）

    FIntPoint UpdateAtlasSize;  // 今フレームの更新アトラスサイズ
    uint32 MaxUpdateTiles;      // 最大タイル数
    uint32 UpdateFactor;        // 更新レート（全ページのうち何分の1を更新するか）
};
```

---

## FLumenCardTileUpdateContext

Card を **8×8 タイル単位** でバッチ処理するコンテキスト。  
Compute Shader の Dispatch 粒度を Card ページ単位からタイル単位に細分化する。

```cpp
struct FLumenCardTileUpdateContext {
    FRDGBufferRef CardTileAllocator;            // タイル数カウンタ
    FRDGBufferRef CardTiles;                    // タイルインデックス配列
    FRDGBufferRef DispatchCardTilesIndirectArgs; // Dispatch 用 Indirect Args
};
```

---

## シェーダークラス

### FRasterizeToCardsVS

Card アトラスへのラスタライズに使うスクリーン空間 VS。  
各 Card ページを適切な UV 範囲にマッピングして描画する。

```cpp
class FRasterizeToCardsVS : public FGlobalShader {
    DECLARE_GLOBAL_SHADER(FRasterizeToCardsVS)
    SHADER_USE_PARAMETER_STRUCT(FRasterizeToCardsVS, FGlobalShader)

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
        SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint>, CardPageIndexData)
        SHADER_PARAMETER_STRUCT_INCLUDE(FLumenCardScene, LumenCardScene)
    END_SHADER_PARAMETER_STRUCT()
};
```

### FClearLumenCardsPS

新規キャプチャ Card のアトラスを消去する PS。  
ターゲット数（1 or 2）をパーミュテーションで切り替える。

```cpp
class FClearLumenCardsPS : public FGlobalShader {
    DECLARE_GLOBAL_SHADER(FClearLumenCardsPS)

    class FNumTargets : SHADER_PERMUTATION_RANGE_INT("NUM_TARGETS", 1, 2);
    using FPermutationDomain = TShaderPermutationDomain<FNumTargets>;
};
```

### FCopyCardCaptureLightingToAtlasPS

Card キャプチャで取得した照明を Surface Cache アトラスにコピーする PS。  
Indirect Lighting の有無と Resample の要否をパーミュテーションで制御。

```cpp
class FCopyCardCaptureLightingToAtlasPS : public FGlobalShader {
    DECLARE_GLOBAL_SHADER(FCopyCardCaptureLightingToAtlasPS)

    class FIndirectLighting : SHADER_PERMUTATION_BOOL("INDIRECT_LIGHTING");
    class FResample        : SHADER_PERMUTATION_BOOL("RESAMPLE");
    using FPermutationDomain = TShaderPermutationDomain<FIndirectLighting, FResample>;
};
```

---

## ELumenLightType

Lumen の Direct Lighting が扱うライトタイプ列挙。

```cpp
enum class ELumenLightType {
    Directional,  // 平行光源
    Point,        // 点光源
    Spot,         // スポットライト
    Rect,         // 矩形ライト
    MAX
};
```

---

## FLumenDirectLightingStochasticData

確率的 Direct Lighting の中間データを保持する構造体。

```cpp
struct FLumenDirectLightingStochasticData {
    FRDGBufferRef  CompactedLightSampleData;      // コンパクトされたライトサンプルデータ
    FRDGBufferRef  CompactedLightSampleAllocator; // サンプル数カウンタ
    FRDGTextureRef SceneDataTexture;              // GBuffer 参照テクスチャ
    FRDGTextureRef LightSamples;                  // サンプリングされた照明テクスチャ
};
```

---

## Lumen 名前空間（LumenSceneLighting.h で宣言）

| 関数 | 説明 |
|-----|------|
| `SetDirectLightingDeferredLightUniformBuffer(...)` | Deferred Light UB を Direct Lighting 用にセット |
| `CombineLumenSceneLighting(...)` | Direct + Indirect をアトラスに合成 |
| `BuildCardUpdateContext(GraphBuilder, Scene, Views, FrameTemporaries, UpdateFactor, Context)` | Card 更新バッチを構築（更新対象ページのリストアップ）|
| `SpliceCardPagesIntoTiles(GraphBuilder, CardUpdateContext, CardTileUpdateContext)` | ページをタイルに細分化 |
| `GetDirectLightingAtlasFormat()` | Direct Lighting アトラスのピクセルフォーマット |
| `GetIndirectLightingAtlasFormat()` | Indirect Lighting アトラスのピクセルフォーマット |

---

## LumenSceneDirectLighting 名前空間

Direct Lighting 計算に関するユーティリティ関数群。

```cpp
namespace LumenSceneDirectLighting {
    // バッチ処理可能なライトタイプ数（Point / Spot / Rect）
    constexpr uint32 NumBatchableLightTypes = 3;

    // シャドウレイのバイアス値（ソースごとに独立）
    float GetMeshSDFShadowRayBias();          // Mesh SDF 用
    float GetHeightfieldShadowRayBias();      // Heightfield 用
    float GetGlobalSDFShadowRayBias();        // Global SDF 用
    float GetHardwareRayTracingShadowRayBias(); // HW RT 用

    // 確率的ライティングを使用するか
    bool UseStochasticLighting(const FSceneViewFamily& ViewFamily);

    // ライトタイプごとにタイルを分けるか
    bool UseLightTilesPerLightType();

    // HW RT Direct Lighting シャドウのエントリポイント
    void TraceLumenHardwareRayTracedDirectLightingShadows(...);
    FRDGBufferSRVRef TraceLumenHardwareRayTracedDebug(...);
}
```

---

## 主要 CVar（LumenSceneLighting.cpp）

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.Lighting.ForceLightingUpdate` | 0 | 毎フレーム全 Card を強制更新（デバッグ用）|
| `r.LumenScene.Lighting.Feedback` | 1 | フィードバックバッファの利用有無 |
| `r.LumenScene.DirectLighting.UpdateFactor` | 32 | Direct Lighting の更新レート（1/32 ずつ更新）|
| `r.LumenScene.Radiosity.UpdateFactor` | 64 | Radiosity の更新レート（1/64 ずつ更新）|

---

## Card ライティング更新フロー

```
FDeferredShadingRenderer::RenderLumenSceneLighting()
  │
  ├─ BuildCardUpdateContext()           ← 更新対象 Card ページのリストアップ
  ├─ SpliceCardPagesIntoTiles()         ← ページ→タイルに細分化
  │
  ├─ [Direct Lighting]
  │   ├─ LumenSceneDirectLighting::RenderDirectLighting()
  │   │   ├─ UseStochasticLighting() ? Stochastic : Standard
  │   │   ├─ TraceShadows (SDF / HW RT)
  │   │   └─ Composite → Direct Lighting Atlas
  │   └─ UpdateCardUpdateContext()
  │
  ├─ [Radiosity / Indirect Lighting]
  │   ├─ LumenRadiosity::RenderRadiosity()
  │   └─ Composite → Indirect Lighting Atlas
  │
  └─ CombineLumenSceneLighting()        ← Final Color Atlas へ合成
```
