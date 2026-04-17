# ref: FTranslucencyLightingVolumeTextures / FTranslucentSelfShadowUniformParameters

- 対象ファイル: `TranslucentLightingShaders.h/.cpp` / `ShadowRendering.h`
- 概要: [[20_translucency_overview]]

---

## FTranslucencyLightingVolumeTextures（TranslucentLightingShaders.h）

半透明オブジェクトへの間接照明を格納する 3D テクスチャの組。  
GBuffer が使えない半透明シェーダーが描画時にサンプルして間接光を得る。

```cpp
struct FTranslucencyLightingVolumeTextures
{
    // カスケード × 2 チャンネル（Ambient + Directional）
    // TVC_MAX = TranslucencyVolumeCascadeCount（通常 2）

    FRDGTextureMSAA Ambient[TVC_MAX];
    // フォーマット: R11G11B10F
    // 内容: 等方性 SH 係数（SH L0 相当）
    // Cascade 0: 近距離（高密度、ボクセル細かい）
    // Cascade 1: 遠距離（低密度、ボクセル粗い）

    FRDGTextureMSAA Directional[TVC_MAX];
    // フォーマット: R11G11B10F
    // 内容: 方向性 SH 係数（SH L1 相当）
    // 法線方向との内積で方向性を再現

    // デフォルト解像度: 64×64×64 ボクセル
    // CVar: r.TranslucencyLightingVolumeDim
};
```

---

## TranslucencyVolume の座標系

```
World → Volume UV 変換:
  Volume UV = (WorldPos - VolumeOrigin) / VolumeExtent * 0.5 + 0.5

カスケード分割:
  Cascade 0: カメラ近傍（0 〜 r.TranslucencyLightingVolumeFrustumLength の 1/2）
  Cascade 1: 遠距離（0 〜 r.TranslucencyLightingVolumeFrustumLength）

ヘルパー関数（TranslucentLightingShaders.usf）:
  float3 TranslucencyVolumeUV(float3 WorldPos, int CascadeIndex)
    → ワールド座標からカスケード内の UV を返す
```

---

## SH 照明の格納方式

```
SH（Spherical Harmonics）L0 / L1 展開:

Ambient[Cascade].rgb   = Irradiance × SH_L0 係数（= 1 / sqrt(4π)）
  → 全方向均等な照明成分

Directional[Cascade].rgb = Irradiance × SH_L1 係数（= 法線依存の 3 成分）
  → Directional[Cascade].r ∝ SH.L1.x（X方向成分）
  → Directional[Cascade].g ∝ SH.L1.y（Y方向成分）
  → Directional[Cascade].b ∝ SH.L1.z（Z方向成分）

復元:
  IndirectLight = AmbientSH
               + dot(Normal, DirectionalSH.rgb) × DirectionalSH
  SceneColor += IndirectLight × BaseColor
```

---

## InjectTranslucencyLightingVolumeParameters

```cpp
// TranslucentLightingShaders.h（注入 CS のパラメータ）
BEGIN_SHADER_PARAMETER_STRUCT(FTranslucencyLightingVolumeParameters, )
    SHADER_PARAMETER(FVector3f, VolumeBoxMin)   // ボリュームの最小コーナー
    SHADER_PARAMETER(FVector3f, VolumeBoxMax)   // ボリュームの最大コーナー
    SHADER_PARAMETER(int32, VolumeCascadeIndex) // 0 or 1
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture3D<float4>, RWAmbient)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture3D<float4>, RWDirectional)
END_SHADER_PARAMETER_STRUCT()
```

---

## FTranslucentSelfShadowUniformParameters（ShadowRendering.h）

半透明オブジェクトのセルフシャドウ（Translucent Shadow）用パラメータ。  
`TShadowProjectionFromTranslucencyPS` がこれを使ってシャドウを合成する。

```cpp
BEGIN_UNIFORM_BUFFER_STRUCT(FTranslucentSelfShadowUniformParameters, )
    // 半透明シャドウマップのサンプリングに必要なパラメータ
    SHADER_PARAMETER(FMatrix44f, WorldToShadowMatrix)    // World → Shadow UV
    SHADER_PARAMETER(FVector4f, ShadowUVMinMax)          // Shadow Atlas 内クランプ範囲
    SHADER_PARAMETER(FLinearColor, DirectionalLightColor) // 方向光の色
    SHADER_PARAMETER(FVector3f, DirectionalLightDirection)
    SHADER_PARAMETER_TEXTURE(Texture2D, ShadowDepthTexture)   // Shadow Depth Atlas
    SHADER_PARAMETER_SAMPLER(SamplerState, ShadowDepthSampler)
    SHADER_PARAMETER_TEXTURE(Texture2D, BackfaceShadowDepthTexture) // 裏面シャドウ
    SHADER_PARAMETER_SAMPLER(SamplerState, BackfaceShadowDepthSampler)
END_UNIFORM_BUFFER_STRUCT()

// デフォルト（シャドウなし）用グローバルバッファ
extern TGlobalResource<FEmptyTranslucentSelfShadowUniformBuffer>
    GEmptyTranslucentSelfShadowUniformBuffer;
```

---

## Lumen TranslucencyVolume パラメータ

```cpp
// LumenTranslucencyVolume.h（r.Lumen.Translucency=1 時）
struct FLumenTranslucencyLightingParameters
{
    // Lumen Radiance Cache からサンプルするためのパラメータ
    FRDGTextureRef RadianceCacheAtlas;         // Irradiance Probe アトラス
    FRDGBufferRef  RadianceCacheProbeTileData; // プローブタイルデータ
    FVector3f      VolumeOrigin;
    float          VolumeExtent;
    int32          ProbeSpacingInRadianceCacheTexels;
    // → TranslucencyVolume の代わりに Lumen Radiance Cache をサンプル
    // → 精度は高いが計算コストが増大
};
```

---

## ClearTranslucencyLightingVolumeTextures

```
毎フレームの初期化:
ClearTranslucencyLightingVolumeTextures()
  → Ambient[0], Ambient[1], Directional[0], Directional[1] を 0 クリア
  → RDG ClearUAV パスとして発行

注入後（InjectTranslucencyLightingVolume）:
  → Ambient / Directional に SH 照明が書き込まれる

フィルタリング後（FilterTranslucencyVolume）:
  → 3D Gaussian ブラー（X/Y/Z 3 パス）でボクセル境界ノイズを除去
  → r.TranslucencyVolumeBlur=1 の場合のみ実行
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.TranslucencyLightingVolumeDim` | 64 | ボリューム解像度 |
| `r.TranslucencyVolumeBlur` | 1 | 3D Gaussian ブラー有効 |
| `r.TranslucencyLightingVolumeFrustumLength` | 8000 | カスケード 0 の最大距離 |
| `r.Lumen.Translucency` | 0 | Lumen TranslucencyVolume 置換 |

---

## 関連リファレンス

- [[ref_translucent_rendering]] — `FTranslucencyPassResources` / `FTranslucentPrimSet`
- [[b_translucent_lighting]] — TranslucencyVolume 照明注入フロー
