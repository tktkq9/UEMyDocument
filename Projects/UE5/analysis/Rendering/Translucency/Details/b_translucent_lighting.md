# b: Translucency 照明（TranslucencyVolume）

- 対象ファイル: `TranslucentLightingShaders.h/.cpp`
- 概要: [[20_translucency_overview]]

---

## 概要

半透明オブジェクトは GBuffer を使えないため、  
**TranslucencyVolume**（3D テクスチャ）に事前に照明情報を焼き込み、  
描画時にピクセルのワールド座標からサンプルして間接光として適用する。

---

## TranslucencyVolume の構造

```cpp
// TranslucentLightingShaders.h（概略）
struct FTranslucencyLightingVolumeTextures
{
    // カスケード × 2チャンネル（Ambient + Directional）
    FRDGTextureMSAA Ambient[TVC_MAX];      // R11G11B10F: アンビエント SH 係数
    FRDGTextureMSAA Directional[TVC_MAX];  // R11G11B10F: ディレクショナル SH 係数

    // デフォルトで 64×64×64 ボクセル × カスケード数
    // Cascade 0: 近距離（高密度）
    // Cascade 1: 遠距離（低密度）
    // TVC_MAX = TranslucencyVolumeCascadeCount（通常 2）
};
```

---

## InjectTranslucencyLightingVolume() フロー

```
RenderTranslucencyLightingVolume()              TranslucentLightingShaders.cpp
  │
  ├─ [A] ボリュームテクスチャ初期化
  │   ClearTranslucencyLightingVolumeTextures()
  │   → Ambient / Directional テクスチャをゼロクリア
  │
  ├─ [B] 各ライスト寄与の注入
  │   for each LocalLight in VisibleLocalLights:
  │     InjectTranslucencyLightingVolume()
  │       → 各 Froxel（ボクセル）に対して:
  │           WorldPos = FroxelCenter（カスケードの Z 分布から計算）
  │           LightDir = normalize(LightPos - WorldPos)
  │           Attenuation = InvSquareAttenuation(Distance, InvRadius)
  │           ShadowFactor = ShadowMap.Sample(WorldPos) or 1.0（シャドウなし時）
  │           Irradiance = LightColor × Attenuation × ShadowFactor
  │
  │       → SH（Spherical Harmonics）展開:
  │           Ambient[Cascade].xyz    += Irradiance × SH.L0   (等方性成分)
  │           Directional[Cascade].xyz += Irradiance × SH.L1  (方向性成分)
  │
  ├─ [C] Directional Light 注入
  │   for CSM Cascade:
  │     InjectTranslucencyLightingVolumeCascade()
  │     → CSM の Shadow Map からシャドウを評価
  │     → ボリューム全体に Directional Light を注入
  │
  └─ [D] FilterTranslucencyVolume()
      → 3D Gaussian ブラー（3パス: X, Y, Z 方向）
      → r.TranslucencyVolumeBlur=1 時のみ実行
      → ボクセル境界のアーティファクトを軽減

【半透明シェーダーでのサンプリング】
  BasePassPixelShader.usf（半透明用）:
    float3 WorldPos = GetWorldPosition(ScreenUV, Depth)
    float3 AmbientSH  = TranslucencyAmbient.SampleLevel(
                            TranslucencyVolumeUV(WorldPos, Cascade), 0)
    float3 DirectionalSH = TranslucencyDirectional.SampleLevel(
                            TranslucencyVolumeUV(WorldPos, Cascade), 0)
    float3 IndirectLight = AmbientSH + DirectionalSH × dot(Normal, SH.L1Dir)
    SceneColor += IndirectLight × BaseColor
```

---

## Lumen Translucency Radiance Cache との連携

```
[Lumen 有効時]
Lumen::RenderTranslucencyVolume()               LumenTranslucencyVolume.cpp
  → TranslucencyVolume の代わりに
    Lumen Radiance Cache（Irradiance Probe）からサンプル
  → 精度は高いが計算コスト増（r.Lumen.Translucency=1 で有効）

[フォールバック（Lumen 無効 or 無効設定）]
  通常の InjectTranslucencyLightingVolume() を使用
  → 近似的だが軽量

【MegaLights TranslucencyVolume との関係】
  MegaLights が有効（r.MegaLights.Volume=1）の場合:
  → FMegaLightsVolume.TranslucencyAmbient / Directional が
    TranslucencyLightingVolumeTextures の代わりとして使用される
  → MegaLights 対象外ライスト分は通常ボリュームで補完
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.TranslucencyLightingVolumeDim` | 64 | ボリューム解像度（64×64×64）|
| `r.TranslucencyVolumeBlur` | 1 | 3D Gaussian ブラー有効 |
| `r.TranslucencyLightingVolumeFrustumLength` | 8000 | カスケード 0 の最大距離 |
| `r.Lumen.Translucency` | 0 | Lumen による TranslucencyVolume 置換 |

---

## 関連リファレンス

- [[ref_translucent_lighting]] — `FTranslucencyLightingVolumeTextures` / パラメータ詳細
- [[a_translucent_rendering]] — TranslucencyVolume のサンプリング位置（描画フェーズ）
