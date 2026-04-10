# リファレンス：LumenTranslucencyVolumeLighting.h/cpp / LumenTranslucencyVolumeHardwareRayTracing.cpp

- グループ: d - Radiance Cache
- 上位: [[d_lumen_radiance_cache]]
- 関連: [[ref_lumen_radiance_cache]] | [[ref_lumen_translucency_radiance_cache]] | [[ref_lumen_hwrt_common]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenTranslucencyVolumeLighting.h`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenTranslucencyVolumeLighting.cpp`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenTranslucencyVolumeHardwareRayTracing.cpp`

---

## 概要

透明マテリアル（特にフォグ・ボリュームトレース）への **GI ボリューム照明** を提供するファイル群。  
フラスタム空間の 3D グリッド（フロクセル）に Lumen の間接照明を焼き込み、  
透明 BasePass や VolumetricFog がサンプリングできるようにする。  
Radiance Cache の補間結果を Volume テクスチャに書き込む形で実装される。

---

## FLumenTranslucencyGIVolume

GI ボリュームテクスチャと Radiance Cache パラメータをまとめたクラス。  
`RenderLumenTranslucencyVolumeLighting()` で生成され、透明 BasePass に渡される。

```cpp
class FLumenTranslucencyGIVolume {
public:
    LumenRadianceCache::FRadianceCacheInterpolationParameters RadianceCacheInterpolationParameters;

    FRDGTextureRef Texture0        = nullptr;
    FRDGTextureRef Texture1        = nullptr;
    FRDGTextureRef HistoryTexture0 = nullptr;
    FRDGTextureRef HistoryTexture1 = nullptr;

    FVector   GridZParams       = FVector::ZeroVector;
    uint32    GridPixelSizeShift = 0;
    FIntVector GridSize         = FIntVector::ZeroValue;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `RadianceCacheInterpolationParameters` | `FRadianceCacheInterpolationParameters` | Radiance Cache 補間パラメータ。Volume トレースのヒット後照明サンプリングに使用 |
| `Texture0` | `FRDGTextureRef` | GI ボリューム SH 係数テクスチャ 0（現フレーム）。RGB チャンネルに SH 係数を格納 |
| `Texture1` | `FRDGTextureRef` | GI ボリューム SH 係数テクスチャ 1（現フレーム）。残りの SH 係数 |
| `HistoryTexture0` | `FRDGTextureRef` | 前フレームのテクスチャ 0（テンポラル蓄積でノイズ低減）|
| `HistoryTexture1` | `FRDGTextureRef` | 前フレームのテクスチャ 1 |
| `GridZParams` | `FVector` | フロクセルグリッドの Z 方向パラメータ（近距離 → 遠距離の対数分布）|
| `GridPixelSizeShift` | `uint32` | グリッドセルの画面ピクセルサイズを 2 の冪で表現（シフト量）|
| `GridSize` | `FIntVector` | グリッドサイズ（XYZ 方向のセル数）|

### 使用箇所

- [[ref_lumen_translucency_volume]] — `GetLumenTranslucencyLightingParameters()` の引数として `FLumenTranslucencyLightingParameters` に変換される
- [[ref_lumen_scene_rendering]] — `FSceneTextures` 経由で透明 BasePass シェーダーに渡される

---

## FLumenTranslucencyLightingParameters

透明 BasePass シェーダーが参照する照明パラメータ（シェーダー構造体）。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenTranslucencyLightingParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(LumenRadianceCache::FRadianceCacheInterpolationParameters,
        RadianceCacheInterpolationParameters)
    SHADER_PARAMETER_STRUCT_INCLUDE(FLumenFrontLayerTranslucencyReflectionParameters,
        FrontLayerTranslucencyReflectionParameters)
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D, TranslucencyGIVolume0)
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D, TranslucencyGIVolume1)
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D, TranslucencyGIVolumeHistory0)
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D, TranslucencyGIVolumeHistory1)
    SHADER_PARAMETER_SAMPLER(SamplerState, TranslucencyGIVolumeSampler)
    SHADER_PARAMETER(FVector3f, TranslucencyGIGridZParams)
    SHADER_PARAMETER(uint32,    TranslucencyGIGridPixelSizeShift)
    SHADER_PARAMETER(FIntVector, TranslucencyGIGridSize)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `RadianceCacheInterpolationParameters` | `FRadianceCacheInterpolationParameters` | Radiance Cache 補間パラメータ（透明サーフェスの反射照明）|
| `FrontLayerTranslucencyReflectionParameters` | `FLumenFrontLayerTranslucencyReflectionParameters` | Front Layer Translucency 専用の反射パラメータ |
| `TranslucencyGIVolume0` | `Texture3D` | GI ボリュームテクスチャ 0（現フレーム）|
| `TranslucencyGIVolume1` | `Texture3D` | GI ボリュームテクスチャ 1（現フレーム）|
| `TranslucencyGIVolumeHistory0` | `Texture3D` | GI ボリューム履歴テクスチャ 0（前フレーム）|
| `TranslucencyGIVolumeHistory1` | `Texture3D` | GI ボリューム履歴テクスチャ 1（前フレーム）|
| `TranslucencyGIVolumeSampler` | `SamplerState` | GI ボリューム用サンプラ（バイリニアクランプ）|
| `TranslucencyGIGridZParams` | `FVector3f` | フロクセルグリッドの Z 分布パラメータ（シェーダー内で深度 → グリッド座標変換に使用）|
| `TranslucencyGIGridPixelSizeShift` | `uint32` | グリッドセルのピクセルサイズシフト量 |
| `TranslucencyGIGridSize` | `FIntVector` | グリッドの XYZ セル数 |

### 使用箇所

- [[ref_lumen_translucency_volume]] — `GetLumenTranslucencyLightingParameters()` で `FLumenTranslucencyGIVolume` から構築
- すべての透明 BasePass シェーダー — `FTranslucentBasePassUniformParameters` 経由でバインド
- VolumetricFog シェーダー — `FLumenTranslucencyLightingUniforms` 経由でバインド

---

## GetLumenTranslucencyLightingParameters

`FLumenTranslucencyGIVolume` から `FLumenTranslucencyLightingParameters` を構築するヘルパー関数。

```cpp
extern FLumenTranslucencyLightingParameters GetLumenTranslucencyLightingParameters(
    FRDGBuilder& GraphBuilder,
    const FLumenTranslucencyGIVolume& LumenTranslucencyGIVolume,
    const FLumenFrontLayerTranslucency& LumenFrontLayerTranslucency);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー（nullptr チェックでダミーテクスチャを作成する場合がある）|
| `LumenTranslucencyGIVolume` | `const FLumenTranslucencyGIVolume&` | GI ボリュームデータ（ソース）|
| `LumenFrontLayerTranslucency` | `const FLumenFrontLayerTranslucency&` | Front Layer 反射データ |

**戻り値**: `FLumenTranslucencyLightingParameters` — 構築されたシェーダーパラメータ

### 使用箇所

- [[ref_lumen_scene_rendering]] — 透明 BasePass の UniformBuffer 構築時に呼ばれる
- VolumetricFog パス — `FLumenTranslucencyLightingUniforms` の構築に使用

---

## FLumenTranslucencyLightingUniforms

VolumetricFog や HeterogeneousVolumes がバインドするグローバルユニフォームバッファ。

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FLumenTranslucencyLightingUniforms, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FLumenTranslucencyLightingParameters, Parameters)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

### 使用箇所

- VolumetricFog シェーダー — `FLumenTranslucencyLightingParameters` をグローバル UB としてバインド
- HeterogeneousVolumes — 同様にグローバル UB としてバインド

---

## FLumenTranslucencyLightingVolumeParameters

Volume トレーシング用のパラメータ群。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenTranslucencyLightingVolumeParameters, )
    SHADER_PARAMETER_STRUCT_REF(FBlueNoise, BlueNoise)
    SHADER_PARAMETER(FVector3f, TranslucencyGIGridZParams)
    SHADER_PARAMETER(uint32,    TranslucencyGIGridPixelSizeShift)
    SHADER_PARAMETER(FIntVector, TranslucencyGIGridSize)
    SHADER_PARAMETER(int32,  FroxelDirectionJitterFrameIndex)
    SHADER_PARAMETER(FVector3f, FrameJitterOffset)
    SHADER_PARAMETER(FMatrix44f, UnjitteredClipToTranslatedWorld)
    SHADER_PARAMETER(uint32, TranslucencyVolumeTracingOctahedronResolution)
    SHADER_PARAMETER(float, HZBMipLevel)
    SHADER_PARAMETER(float, GridCenterOffsetFromDepthBuffer)
    SHADER_PARAMETER(float, GridCenterOffsetThresholdToAcceptDepthBufferOffset)
    SHADER_PARAMETER_STRUCT_INCLUDE(FHZBParameters, HZBParameters)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `BlueNoise` | `FBlueNoise` | ブルーノイズテクスチャ（トレース方向ジッターに使用）|
| `TranslucencyGIGridZParams` | `FVector3f` | フロクセルグリッドの Z 分布パラメータ |
| `TranslucencyGIGridPixelSizeShift` | `uint32` | グリッドセルのピクセルサイズシフト量 |
| `TranslucencyGIGridSize` | `FIntVector` | グリッドの XYZ セル数 |
| `FroxelDirectionJitterFrameIndex` | `int32` | フロクセルトレース方向のジッターフレームインデックス（時間的ジッターでノイズ低減）|
| `FrameJitterOffset` | `FVector3f` | フレームごとのワールド空間ジッターオフセット |
| `UnjitteredClipToTranslatedWorld` | `FMatrix44f` | ジッターなしのクリップ空間 → ワールド空間変換行列 |
| `TranslucencyVolumeTracingOctahedronResolution` | `uint32` | 八面体マップのトレース解像度（各フロクセルのサンプル方向数）|
| `HZBMipLevel` | `float` | HZB オクルージョンテスト用ミップレベル |
| `GridCenterOffsetFromDepthBuffer` | `float` | 深度バッファからのグリッドセンターオフセット（cm）|
| `GridCenterOffsetThresholdToAcceptDepthBufferOffset` | `float` | オフセット許容しきい値（大きすぎるオフセットは無視）|
| `HZBParameters` | `FHZBParameters` | HZB テクスチャ・サンプラ設定 |

### 使用箇所

- [[ref_lumen_translucency_volume]] — Volume トレースパスの CS / HW RT シェーダーにバインド
- [[ref_lumen_translucency_volume]] — `HardwareRayTraceTranslucencyVolume()` の引数として渡される

---

## FLumenTranslucencyLightingVolumeTraceSetupParameters

Volume トレースの基本設定パラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenTranslucencyLightingVolumeTraceSetupParameters, )
    SHADER_PARAMETER(float, StepFactor)
    SHADER_PARAMETER(float, MaxTraceDistance)
    SHADER_PARAMETER(float, VoxelTraceStartDistanceScale)
    SHADER_PARAMETER(float, MaxRayIntensity)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `StepFactor` | `float` | SDF / Voxel レイマーチのステップ係数（大きいほど高速だが精度低下）|
| `MaxTraceDistance` | `float` | トレースの最大距離（cm）。これを超えたレイはミスとして扱う |
| `VoxelTraceStartDistanceScale` | `float` | Voxel トレース開始距離のスケール（近距離の Surface Cache との切り替え点）|
| `MaxRayIntensity` | `float` | レイの最大輝度クランプ値（ファイアフライ対策）|

### 使用箇所

- [[ref_lumen_translucency_volume]] — `RenderLumenTranslucencyVolumeLighting()` で CVar から構築
- [[ref_lumen_translucency_volume]] — `HardwareRayTraceTranslucencyVolume()` の引数として渡される

---

## Lumen 名前空間（Translucency Volume 判定）

```cpp
namespace Lumen {
    bool UseHardwareRayTracedTranslucencyVolume(const FSceneViewFamily& ViewFamily);
    bool UseLumenTranslucencyRadianceCacheReflections(const FSceneViewFamily& ViewFamily);
}
```

### メンバ関数

| 関数 | 戻り値 | 説明 |
|------|--------|------|
| `UseHardwareRayTracedTranslucencyVolume()` | `bool` | Translucency Volume の HW RT を使うか。`r.Lumen.TranslucencyVolume.HardwareRayTracing` かつ RT 対応 GPU |
| `UseLumenTranslucencyRadianceCacheReflections()` | `bool` | 透明マテリアルへの Radiance Cache Reflections を使うか（[[ref_lumen_translucency_radiance_cache]] 参照）|

### 使用箇所

- [[ref_lumen_translucency_volume]] — `RenderLumenTranslucencyVolumeLighting()` でトレース方式を選択
- [[ref_lumen_scene_rendering]] — 透明マテリアルのライティングパスを選択する際に参照

---

## LumenTranslucencyVolumeRadianceCache 名前空間

```cpp
namespace LumenTranslucencyVolumeRadianceCache {
    LumenRadianceCache::FRadianceCacheInputs SetupRadianceCacheInputs(const FViewInfo& View);
}
```

### SetupRadianceCacheInputs

| 引数 | 型 | 説明 |
|------|-----|------|
| `View` | `const FViewInfo&` | カメラビュー（CVar の参照先）|

**戻り値**: `LumenRadianceCache::FRadianceCacheInputs` — Translucency Volume 専用の Radiance Cache 設定

**特徴**: GI 用と異なり `CalculateIrradiance = 0`、解像度が低め（コスト削減優先）

### 使用箇所

- [[ref_lumen_translucency_volume]] — `RenderLumenTranslucencyVolumeLighting()` 内で Radiance Cache 更新の入力として使用

---

## HardwareRayTraceTranslucencyVolume

SW RT の代わりに HW RT でフロクセルをトレースする関数。

```cpp
extern void HardwareRayTraceTranslucencyVolume(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FLumenCardTracingParameters& TracingParameters,
    LumenRadianceCache::FRadianceCacheInterpolationParameters RadianceCacheParameters,
    FLumenTranslucencyLightingVolumeParameters VolumeParameters,
    FLumenTranslucencyLightingVolumeTraceSetupParameters TraceSetupParameters,
    FRDGTextureRef VolumeTraceRadiance,
    FRDGTextureRef VolumeTraceHitDistance,
    ERDGPassFlags ComputePassFlags);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `View` | `const FViewInfo&` | カメラビュー（TLAS バインドに使用）|
| `TracingParameters` | `const FLumenCardTracingParameters&` | Surface Cache アトラス・トレース共通パラメータ（[[ref_lumen_tracing_utils]] 参照）|
| `RadianceCacheParameters` | `FRadianceCacheInterpolationParameters` | ヒット時の照明サンプル用 Radiance Cache |
| `VolumeParameters` | `FLumenTranslucencyLightingVolumeParameters` | フロクセルグリッド設定・ジッターパラメータ |
| `TraceSetupParameters` | `FLumenTranslucencyLightingVolumeTraceSetupParameters` | ステップ係数・最大距離等のトレース設定 |
| `VolumeTraceRadiance` | `FRDGTextureRef` | 出力: トレースした Radiance（3D テクスチャ UAV）|
| `VolumeTraceHitDistance` | `FRDGTextureRef` | 出力: ヒット距離（3D テクスチャ UAV。テンポラル蓄積の重み計算用）|
| `ComputePassFlags` | `ERDGPassFlags` | AsyncCompute / Compute の選択フラグ |

### 内部処理フロー

1. **フロクセルグリッドに対して RT Dispatch**
   ```cpp
   // グリッドサイズ × 八面体解像度 スレッドをディスパッチ
   // 各スレッドが 1 フロクセル × 1 トレース方向を担当
   ```

2. **HW RT レイ発射**
   ```cpp
   // FLumenHardwareRayTracingShaderBase 派生シェーダーを使用
   // LumenMinimal ペイロードでヒット座標を取得
   ```

3. **ヒット時の照明サンプリング**
   ```cpp
   // ヒット点の Surface Cache (FinalLightingAtlas) をサンプリング
   // または RadianceCacheParameters から Radiance Cache を補間
   ```

4. **VolumeTraceRadiance / VolumeTraceHitDistance に書き込み**
   ```cpp
   // 後続の SH 射影パスへ受け渡す
   ```

### 使用箇所

- [[ref_lumen_translucency_volume]] — `RenderLumenTranslucencyVolumeLighting()` 内で `UseHardwareRayTracedTranslucencyVolume()` が true の場合に呼ばれる

---

## Translucency GI ボリュームの更新フロー

```
RenderLumenTranslucencyVolumeLighting()
  │
  ├─ フロクセルグリッドのサイズ決定
  │   → GridSize / GridZParams / GridPixelSizeShift を計算
  │
  ├─ [Radiance Cache 更新]
  │   → LumenTranslucencyVolumeRadianceCache::SetupRadianceCacheInputs()
  │   → UpdateRadianceCaches() でプローブをトレース
  │
  ├─ [Volume トレース]
  │   ├─ SW RT: 各フロクセルから Surface Cache をトレース
  │   └─ HW RT: HardwareRayTraceTranslucencyVolume() を呼ぶ
  │      → VolumeTraceRadiance / VolumeTraceHitDistance に書き込み
  │
  ├─ [SH への射影]
  │   → フロクセルの Radiance → SH 係数 → Texture0 / Texture1
  │
  ├─ [テンポラル蓄積]
  │   → HistoryTexture0/1 と現フレームを混合してノイズ低減
  │   → VolumeTraceHitDistance を重みとして使用（遠距離ヒットほど再利用を増やす）
  │
  └─ FLumenTranslucencyGIVolume を透明 BasePass / VolumetricFog に渡す
```
