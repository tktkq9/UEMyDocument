# リファレンス：LumenRadianceCacheInternal.h / LumenRadianceCacheInterpolation.h

- グループ: d - Radiance Cache
- 上位: [[d_lumen_radiance_cache]]
- 関連: [[ref_lumen_radiance_cache]] | [[ref_lumen_view_state]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenRadianceCacheInternal.h`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenRadianceCacheInterpolation.h`

---

## 概要

Radiance Cache の**内部実装データ構造**と**補間シェーダーパラメータ**を定義するヘッダ群。  
`LumenRadianceCacheInterpolation.h` は他システムが参照する公開型、  
`LumenRadianceCacheInternal.h` は `LumenRadianceCache.cpp` 内部専用の実装型を持つ。

---

## LumenRadianceCacheInterpolation.h

### 定数

```cpp
namespace LumenRadianceCache {
    static constexpr int32 MaxClipmaps = 6;              // 最大クリップマップ数
    static constexpr int32 MinRadianceProbeResolution = 8;
}
```

---

### FRadianceCacheInputs

Radiance Cache の**設定パラメータ**（シェーダーパラメータ構造体）。  
`UpdateRadianceCaches()` に入力として渡し、シェーダー内でも参照される。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FRadianceCacheInputs, )
    SHADER_PARAMETER(float,    ReprojectionRadiusScale)     // リプロジェクション半径スケール（大きいほど遠くのキャッシュを使用）
    SHADER_PARAMETER(float,    ClipmapWorldExtent)          // 最初のクリップマップの半径（cm）
    SHADER_PARAMETER(float,    ClipmapDistributionBase)     // クリップマップ間のサイズ比（Pow のベース）
    SHADER_PARAMETER(float,    InvClipmapFadeSize)          // クリップマップ境界フェードの逆数
    SHADER_PARAMETER(float,    ProbeTMinScale)              // プローブのトレース開始距離スケール
    SHADER_PARAMETER(FIntPoint, ProbeAtlasResolutionInProbes) // プローブアトラスのプローブ数（2D）
    SHADER_PARAMETER(uint32,   RadianceProbeClipmapResolution) // クリップマップ 1 辺あたりのプローブ数
    SHADER_PARAMETER(uint32,   NumRadianceProbeClipmaps)    // 使用クリップマップ数（≤ MaxClipmaps）
    SHADER_PARAMETER(uint32,   RadianceProbeResolution)     // プローブのトレース解像度
    SHADER_PARAMETER(uint32,   FinalProbeResolution)        // フィルタ後のプローブ解像度
    SHADER_PARAMETER(uint32,   FinalRadianceAtlasMaxMip)    // Radiance アトラスの最大ミップ
    SHADER_PARAMETER(uint32,   CalculateIrradiance)         // 照度（Irradiance）を計算するか
    SHADER_PARAMETER(uint32,   UseSkyVisibility)            // Sky Visibility を使うか
    SHADER_PARAMETER(uint32,   IrradianceProbeResolution)   // 照度プローブの解像度
    SHADER_PARAMETER(uint32,   OcclusionProbeResolution)    // オクルージョンプローブの解像度
    SHADER_PARAMETER(uint32,   NumProbesToTraceBudget)      // 1フレームに更新するプローブ数の上限
    SHADER_PARAMETER(uint32,   RadianceCacheStats)          // 統計収集フラグ（デバッグ）
END_SHADER_PARAMETER_STRUCT()

// デフォルト値で初期化した FRadianceCacheInputs を返す
FRadianceCacheInputs GetDefaultRadianceCacheInputs();
```

---

### FRadianceCacheInterpolationParameters

Radiance Cache から補間して照明を取得するためのシェーダーパラメータ。  
Screen Probe・Translucency など、補間側のすべてのシェーダーが使用する。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FRadianceCacheInterpolationParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FRadianceCacheInputs, RadianceCacheInputs)

    // クリップマップ単位のインディレクションテクスチャ（プローブ → アトラス座標）
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D<uint>,   RadianceProbeIndirectionTexture)

    // プローブアトラス（各種データ）
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float3>, RadianceCacheFinalRadianceAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float>,  RadianceCacheFinalSkyVisibilityAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float3>, RadianceCacheFinalIrradianceAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float2>, RadianceCacheProbeOcclusionAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float>,  RadianceCacheDepthAtlas)

    // プローブのワールド座標オフセット（SnapAndReprojection）
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, ProbeWorldOffset)

    // クリップマップごとの設定（X=TMin, Y=WorldToProbeCoordScale, Z=ProbeCoordToWorldScale）
    SHADER_PARAMETER_ARRAY(FVector4f, RadianceProbeSettings,      [MaxClipmaps])
    // クリップマップごとのコーナー位置とセルサイズ（XYZ=コーナー, W=セルサイズ）
    SHADER_PARAMETER_ARRAY(FVector4f, ClipmapCornerTWSAndCellSize, [MaxClipmaps])

    // アトラス解像度の逆数（UV 計算用）
    SHADER_PARAMETER(FVector2f, InvProbeFinalRadianceAtlasResolution)
    SHADER_PARAMETER(FVector2f, InvProbeFinalIrradianceAtlasResolution)
    SHADER_PARAMETER(FVector2f, InvProbeDepthAtlasResolution)

    SHADER_PARAMETER(float,  RadianceCacheOneOverCachedLightingPreExposure)
    SHADER_PARAMETER(uint32, OverrideCacheOcclusionLighting)
    SHADER_PARAMETER(uint32, ShowBlackRadianceCacheLighting)
    SHADER_PARAMETER(uint32, ProbeAtlasResolutionModuloMask)
    SHADER_PARAMETER(uint32, ProbeAtlasResolutionDivideShift)
END_SHADER_PARAMETER_STRUCT()
```

### セッタ関数

```cpp
// クリップマップ Index の TMin 値をセット
inline void SetRadianceProbeClipmapTMin(
    FRadianceCacheInterpolationParameters& Params, uint32 Index, float Value);

// クリップマップ Index のコーナーワールド位置をセット
inline void SetClipmapCornerTWS(
    FRadianceCacheInterpolationParameters& Params, uint32 Index, FVector3f Corner);

// クリップマップ Index のセルサイズをセット
inline void SetClipmapCellSize(
    FRadianceCacheInterpolationParameters& Params, uint32 Index, float CellSize);

// FRadianceCacheState から FRadianceCacheInterpolationParameters を構築
void GetInterpolationParameters(
    const FViewInfo& View,
    FRDGBuilder& GraphBuilder,
    const FRadianceCacheState& RadianceCacheState,
    const LumenRadianceCache::FRadianceCacheInputs& RadianceCacheInputs,
    FRadianceCacheInterpolationParameters& OutParameters);
```

---

## LumenRadianceCacheInternal.h

### 定数

```cpp
namespace LumenRadianceCache {
    const int32 TRACE_TILE_SIZE_2D = 8;               // トレースタイルサイズ（8×8）
    const int32 TRACE_TILE_ATLAS_STRITE_IN_TILES = 512; // アトラス幅（タイル単位）
}
```

---

### FRadianceCacheSetup

Radiance Cache 更新フレームの**一時セットアップデータ**。  
`UpdateRadianceCaches()` 内部で生成され、HW RT トレース関数に渡される。

```cpp
class FRadianceCacheSetup {
public:
    TArray<FRadianceCacheClipmap> LastFrameClipmaps; // 前フレームのクリップマップ状態

    // フレーム内の一時 RDG テクスチャ
    FRDGTextureRef DepthProbeAtlasTexture;           // プローブ深度アトラス
    FRDGTextureRef FinalIrradianceAtlas;             // 照度アトラス（最終結果）
    FRDGTextureRef ProbeOcclusionAtlas;              // プローブオクルージョン
    FRDGTextureRef FinalRadianceAtlas;               // Radiance アトラス（最終結果）
    FRDGTextureRef FinalSkyVisibilityAtlas;          // Sky Visibility アトラス
    FRDGTextureRef RadianceProbeAtlasTextureSource;  // トレース元 Radiance アトラス
    FRDGTextureRef SkyVisibilityProbeAtlasTextureSource;

    bool bPersistentCache; // 永続キャッシュ（テンポラル再利用）が有効か
};
```

---

## インディレクションテクスチャの仕組み

```
RadianceProbeIndirectionTexture (Texture3D<uint>):
  ├─ 次元: (ClipmapResolution × NumClipmaps, ClipmapResolution, ClipmapResolution)
  └─ 各セルの値: プローブアトラス内のプローブインデックス（UINT32_MAX = 未割り当て）

シェーダー側の補間フロー:
  1. ワールド座標 → クリップマップ座標に変換
  2. RadianceProbeIndirectionTexture をルックアップ → プローブ ID 取得
  3. プローブ ID → アトラス UV に変換
  4. FinalRadianceAtlas / FinalIrradianceAtlas をサンプリング
  5. 隣接 8 プローブから三線形補間
```
