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

### 使用箇所（定数）

- [[ref_lumen_radiance_cache]] — `FRadianceCacheMarkParameters` の配列サイズ `[MaxClipmaps]` に使用
- [[ref_lumen_radiance_cache_internal]] — `FRadianceCacheInterpolationParameters` の配列サイズに使用

---

### FRadianceCacheInputs

Radiance Cache の**設定パラメータ**（シェーダーパラメータ構造体）。  
`UpdateRadianceCaches()` に入力として渡し、シェーダー内でも参照される。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FRadianceCacheInputs, )
    SHADER_PARAMETER(float,    ReprojectionRadiusScale)
    SHADER_PARAMETER(float,    ClipmapWorldExtent)
    SHADER_PARAMETER(float,    ClipmapDistributionBase)
    SHADER_PARAMETER(float,    InvClipmapFadeSize)
    SHADER_PARAMETER(float,    ProbeTMinScale)
    SHADER_PARAMETER(FIntPoint, ProbeAtlasResolutionInProbes)
    SHADER_PARAMETER(uint32,   RadianceProbeClipmapResolution)
    SHADER_PARAMETER(uint32,   NumRadianceProbeClipmaps)
    SHADER_PARAMETER(uint32,   RadianceProbeResolution)
    SHADER_PARAMETER(uint32,   FinalProbeResolution)
    SHADER_PARAMETER(uint32,   FinalRadianceAtlasMaxMip)
    SHADER_PARAMETER(uint32,   CalculateIrradiance)
    SHADER_PARAMETER(uint32,   UseSkyVisibility)
    SHADER_PARAMETER(uint32,   IrradianceProbeResolution)
    SHADER_PARAMETER(uint32,   OcclusionProbeResolution)
    SHADER_PARAMETER(uint32,   NumProbesToTraceBudget)
    SHADER_PARAMETER(uint32,   RadianceCacheStats)
END_SHADER_PARAMETER_STRUCT()

FRadianceCacheInputs GetDefaultRadianceCacheInputs();
```

### メンバ変数（FRadianceCacheInputs）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `ReprojectionRadiusScale` | `float` | リプロジェクション半径スケール。大きいほど遠くのキャッシュを参照（テンポラル再利用拡大）|
| `ClipmapWorldExtent` | `float` | 最初のクリップマップの半径（cm）。例: 5000.0cm |
| `ClipmapDistributionBase` | `float` | クリップマップ間のサイズ比（Pow のベース）。2.0 なら各段が 2 倍に拡大 |
| `InvClipmapFadeSize` | `float` | クリップマップ境界フェードサイズの逆数。境界付近のブレンドに使用 |
| `ProbeTMinScale` | `float` | プローブのトレース開始距離スケール。自己交差回避のための最小距離 |
| `ProbeAtlasResolutionInProbes` | `FIntPoint` | プローブアトラスの 2D サイズ（プローブ数単位）|
| `RadianceProbeClipmapResolution` | `uint32` | クリップマップ 1 辺あたりのプローブ数（例: 32 → 32³ グリッド）|
| `NumRadianceProbeClipmaps` | `uint32` | 使用クリップマップ数（≤ MaxClipmaps=6）|
| `RadianceProbeResolution` | `uint32` | プローブのトレース解像度（1 プローブあたりの方向サンプル数）|
| `FinalProbeResolution` | `uint32` | フィルタ後のプローブ解像度（RadianceProbeResolution + ボーダー）|
| `FinalRadianceAtlasMaxMip` | `uint32` | Radiance アトラスの最大ミップレベル |
| `CalculateIrradiance` | `uint32` | 照度（Irradiance = 方向平均 Radiance）を計算するか（bool として使用）|
| `UseSkyVisibility` | `uint32` | Sky Visibility プローブを生成するか（bool として使用）|
| `IrradianceProbeResolution` | `uint32` | 照度プローブの解像度（FinalProbeResolution より小さい）|
| `OcclusionProbeResolution` | `uint32` | オクルージョンプローブの解像度 |
| `NumProbesToTraceBudget` | `uint32` | 1 フレームに更新するプローブ数の上限。小さいほど負荷が低いがキャッシュの収束が遅い |
| `RadianceCacheStats` | `uint32` | GPU 統計収集フラグ（デバッグ用。0=無効）|

### 使用箇所（FRadianceCacheInputs）

- [[ref_lumen_radiance_cache]] — `FUpdateInputs::RadianceCacheInputs` として格納
- [[ref_lumen_radiance_cache_internal]] — `FRadianceCacheInterpolationParameters` にインクルードされる
- [[ref_lumen_irradiance_field]] — `LumenIrradianceFieldGather::SetupRadianceCacheInputs()` が生成して渡す
- [[ref_lumen_translucency_volume]] — `LumenTranslucencyVolumeRadianceCache::SetupRadianceCacheInputs()` が生成

---

### FRadianceCacheInterpolationParameters

Radiance Cache から補間して照明を取得するためのシェーダーパラメータ。  
Screen Probe・Translucency など、補間側のすべてのシェーダーが使用する。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FRadianceCacheInterpolationParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FRadianceCacheInputs, RadianceCacheInputs)
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D<uint>,   RadianceProbeIndirectionTexture)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float3>, RadianceCacheFinalRadianceAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float>,  RadianceCacheFinalSkyVisibilityAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float3>, RadianceCacheFinalIrradianceAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float2>, RadianceCacheProbeOcclusionAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float>,  RadianceCacheDepthAtlas)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, ProbeWorldOffset)
    SHADER_PARAMETER_ARRAY(FVector4f, RadianceProbeSettings,      [MaxClipmaps])
    SHADER_PARAMETER_ARRAY(FVector4f, ClipmapCornerTWSAndCellSize, [MaxClipmaps])
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

### メンバ変数（FRadianceCacheInterpolationParameters）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `RadianceCacheInputs` | `FRadianceCacheInputs` | 設定パラメータ（クリップマップ数・解像度など）|
| `RadianceProbeIndirectionTexture` | `Texture3D<uint>` | クリップマップ → プローブアトラスへのインディレクション 3D テクスチャ |
| `RadianceCacheFinalRadianceAtlas` | `Texture2D<float3>` | フィルタ済み Radiance アトラス（最終結果）|
| `RadianceCacheFinalSkyVisibilityAtlas` | `Texture2D<float>` | Sky Visibility アトラス（UseSkyVisibility=1 の場合のみ有効）|
| `RadianceCacheFinalIrradianceAtlas` | `Texture2D<float3>` | 照度アトラス（CalculateIrradiance=1 の場合のみ有効）|
| `RadianceCacheProbeOcclusionAtlas` | `Texture2D<float2>` | プローブオクルージョン（Chebyshev 距離 + 二乗距離）|
| `RadianceCacheDepthAtlas` | `Texture2D<float>` | プローブ深度アトラス（自己交差回避の TMin 計算用）|
| `ProbeWorldOffset` | `StructuredBuffer<float4>` | プローブのワールド座標オフセット（Snap & Reprojection 補正用）|
| `RadianceProbeSettings[MaxClipmaps]` | `FVector4f[]` | クリップマップごとの設定（X=TMin, Y=WorldToProbeCoordScale, Z=ProbeCoordToWorldScale）|
| `ClipmapCornerTWSAndCellSize[MaxClipmaps]` | `FVector4f[]` | クリップマップのコーナーワールド座標（XYZ）とセルサイズ（W）|
| `InvProbeFinalRadianceAtlasResolution` | `FVector2f` | Radiance アトラス解像度の逆数（UV 計算用）|
| `InvProbeFinalIrradianceAtlasResolution` | `FVector2f` | 照度アトラス解像度の逆数 |
| `InvProbeDepthAtlasResolution` | `FVector2f` | 深度アトラス解像度の逆数 |
| `RadianceCacheOneOverCachedLightingPreExposure` | `float` | プローブ更新フレームの Pre-Exposure 逆数。補間時に現フレームの露出に合わせるための係数 |
| `OverrideCacheOcclusionLighting` | `uint32` | オクルージョン照明を上書きするか（デバッグ用 bool）|
| `ShowBlackRadianceCacheLighting` | `uint32` | キャッシュ照明を黒にするか（デバッグ用 bool）|
| `ProbeAtlasResolutionModuloMask` | `uint32` | プローブアトラス幅のビットマスク（2の冪アトラスの高速剰余演算用）|
| `ProbeAtlasResolutionDivideShift` | `uint32` | プローブアトラス幅のシフト量（2の冪アトラスの高速除算用）|

### 使用箇所（FRadianceCacheInterpolationParameters）

- [[ref_lumen_radiance_cache]] — `FUpdateOutputs::RadianceCacheParameters` として更新後に格納
- [[ref_lumen_screen_probe_gather]] — Screen Probe トレースシェーダーにバインドして補間
- [[ref_lumen_translucency_radiance_cache]] — 透明サーフェスの反射照明計算に使用
- [[ref_lumen_translucency_volume]] — `FLumenTranslucencyGIVolume::RadianceCacheInterpolationParameters` として格納

---

### GetInterpolationParameters

`FRadianceCacheState` から `FRadianceCacheInterpolationParameters` を構築する関数。

```cpp
void GetInterpolationParameters(
    const FViewInfo& View,
    FRDGBuilder& GraphBuilder,
    const FRadianceCacheState& RadianceCacheState,
    const LumenRadianceCache::FRadianceCacheInputs& RadianceCacheInputs,
    FRadianceCacheInterpolationParameters& OutParameters);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `View` | `const FViewInfo&` | カメラビュー（Pre-Exposure 取得など）|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー（テクスチャ参照の RDG 変換に使用）|
| `RadianceCacheState` | `const FRadianceCacheState&` | フレーム間で保持された永続状態（アトラステクスチャを含む）|
| `RadianceCacheInputs` | `const FRadianceCacheInputs&` | クリップマップ設定（クリップマップ数・解像度など）|
| `OutParameters` | `FRadianceCacheInterpolationParameters&` | 出力先パラメータ構造体 |

### 内部処理フロー

1. **アトラステクスチャの RDG 変換**
   ```cpp
   // FRadianceCacheState の永続テクスチャを RDG テクスチャとして登録
   OutParameters.RadianceCacheFinalRadianceAtlas = GraphBuilder.RegisterExternalTexture(
       RadianceCacheState.FinalRadianceAtlas);
   ```

2. **クリップマップごとの座標変換パラメータをセット**
   ```cpp
   for (int32 ClipmapIndex = 0; ClipmapIndex < NumClipmaps; ++ClipmapIndex) {
       SetClipmapCornerTWS(OutParameters, ClipmapIndex, ClipmapCorner);
       SetClipmapCellSize(OutParameters, ClipmapIndex, CellSize);
       SetRadianceProbeClipmapTMin(OutParameters, ClipmapIndex, TMin);
   }
   ```

3. **Pre-Exposure 係数の設定**
   ```cpp
   // キャッシュ更新時の Pre-Exposure と現フレームの差分を補正
   OutParameters.RadianceCacheOneOverCachedLightingPreExposure =
       1.0f / RadianceCacheState.CachedLightingPreExposure;
   ```

4. **デバッグフラグのコピー**
   ```cpp
   OutParameters.OverrideCacheOcclusionLighting = CVarOverrideCacheOcclusionLighting.GetValueOnRenderThread();
   OutParameters.ShowBlackRadianceCacheLighting  = CVarShowBlackRadianceCacheLighting.GetValueOnRenderThread();
   ```

### 使用箇所

- [[ref_lumen_radiance_cache]] — `UpdateRadianceCaches()` の最終ステップで呼ばれ、`FUpdateOutputs::RadianceCacheParameters` に書き込む

---

### セッタ関数（inline）

> [!note]- SetRadianceProbeClipmapTMin — クリップマップ TMin 値のセット
>
> ```cpp
> inline void SetRadianceProbeClipmapTMin(
>     FRadianceCacheInterpolationParameters& Params,
>     uint32 Index,
>     float Value);
> ```
>
> `RadianceProbeSettings[Index].X` に TMin 値を書き込む。  
> TMin はプローブのレイ開始距離で、自己交差を回避するために使用。
>
> **使用箇所**: [[ref_lumen_radiance_cache_internal]] — `GetInterpolationParameters()` 内のクリップマップ設定ループ

> [!note]- SetClipmapCornerTWS — クリップマップコーナー座標のセット
>
> ```cpp
> inline void SetClipmapCornerTWS(
>     FRadianceCacheInterpolationParameters& Params,
>     uint32 Index,
>     FVector3f Corner);
> ```
>
> `ClipmapCornerTWSAndCellSize[Index].XYZ` にワールドコーナー座標を書き込む。  
> シェーダーがワールド座標 → クリップマップグリッド座標に変換する際に使用。
>
> **使用箇所**: [[ref_lumen_radiance_cache_internal]] — `GetInterpolationParameters()` 内のクリップマップ設定ループ

> [!note]- SetClipmapCellSize — クリップマップセルサイズのセット
>
> ```cpp
> inline void SetClipmapCellSize(
>     FRadianceCacheInterpolationParameters& Params,
>     uint32 Index,
>     float CellSize);
> ```
>
> `ClipmapCornerTWSAndCellSize[Index].W` にセルサイズを書き込む。  
> シェーダーが補間時に隣接プローブへのオフセット計算に使用。
>
> **使用箇所**: [[ref_lumen_radiance_cache_internal]] — `GetInterpolationParameters()` 内のクリップマップ設定ループ

---

## LumenRadianceCacheInternal.h

### 定数

```cpp
namespace LumenRadianceCache {
    const int32 TRACE_TILE_SIZE_2D = 8;               // トレースタイルサイズ（8×8）
    const int32 TRACE_TILE_ATLAS_STRITE_IN_TILES = 512; // アトラス幅（タイル単位）
}
```

### 使用箇所（Internal 定数）

- [[ref_lumen_radiance_cache_hwrt]] — `ProbeTraceTileData` のタイルサイズ（8×8=64 方向）として使用
- [[ref_lumen_radiance_cache]] — トレースタイルバッファのサイズ計算に使用

---

### FRadianceCacheSetup

Radiance Cache 更新フレームの**一時セットアップデータ**。  
`UpdateRadianceCaches()` 内部で生成され、HW RT トレース関数に渡される。

```cpp
class FRadianceCacheSetup {
public:
    TArray<FRadianceCacheClipmap> LastFrameClipmaps; // 前フレームのクリップマップ状態

    // フレーム内の一時 RDG テクスチャ
    FRDGTextureRef DepthProbeAtlasTexture;
    FRDGTextureRef FinalIrradianceAtlas;
    FRDGTextureRef ProbeOcclusionAtlas;
    FRDGTextureRef FinalRadianceAtlas;
    FRDGTextureRef FinalSkyVisibilityAtlas;
    FRDGTextureRef RadianceProbeAtlasTextureSource;
    FRDGTextureRef SkyVisibilityProbeAtlasTextureSource;

    bool bPersistentCache;
};
```

### メンバ変数（FRadianceCacheSetup）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `LastFrameClipmaps` | `TArray<FRadianceCacheClipmap>` | 前フレームのクリップマップ状態（プローブ移動・無効化の差分計算用）|
| `DepthProbeAtlasTexture` | `FRDGTextureRef` | プローブの深度アトラス（オクルージョン計算用）|
| `FinalIrradianceAtlas` | `FRDGTextureRef` | 照度アトラス（CalculateIrradiance=1 の場合に有効）|
| `ProbeOcclusionAtlas` | `FRDGTextureRef` | オクルージョンアトラス（Chebyshev 距離ベースの可視性推定）|
| `FinalRadianceAtlas` | `FRDGTextureRef` | フィルタ済み Radiance アトラス（最終出力）|
| `FinalSkyVisibilityAtlas` | `FRDGTextureRef` | Sky Visibility アトラス |
| `RadianceProbeAtlasTextureSource` | `FRDGTextureRef` | トレース元（未フィルタ）の Radiance アトラス |
| `SkyVisibilityProbeAtlasTextureSource` | `FRDGTextureRef` | トレース元（未フィルタ）の Sky Visibility アトラス |
| `bPersistentCache` | `bool` | テンポラルキャッシュ再利用が有効か（無効時は毎フレーム全プローブ更新）|

### 使用箇所（FRadianceCacheSetup）

- [[ref_lumen_radiance_cache]] — `UpdateRadianceCaches()` 内で各 Radiance Cache 用に 1 つずつ生成
- [[ref_lumen_radiance_cache_hwrt]] — `RenderLumenHardwareRayTracingRadianceCache()` の引数 `SetupOutputArray` として渡される

---

## インディレクションテクスチャの仕組み

```
RadianceProbeIndirectionTexture (Texture3D<uint>):
  ├─ 次元: (ClipmapResolution × NumClipmaps, ClipmapResolution, ClipmapResolution)
  └─ 各セルの値: プローブアトラス内のプローブインデックス（UINT32_MAX = 未割り当て）

シェーダー側の補間フロー:
  1. ワールド座標 → クリップマップ座標に変換
       Coord = (WorldPos - ClipmapCorner) * (1.0 / CellSize)
  2. RadianceProbeIndirectionTexture をルックアップ → プローブ ID 取得
       ProbeIndex = IndirectionTex[ClipmapOffset + Coord]
  3. プローブ ID → アトラス UV に変換
       AtlasUV = (ProbeIndex % AtlasWidth, ProbeIndex / AtlasWidth) / AtlasSize
  4. FinalRadianceAtlas / FinalIrradianceAtlas をサンプリング
  5. 隣接 8 プローブから三線形補間
       → ProbeOcclusionAtlas で遮蔽プローブの寄与を低減
```
