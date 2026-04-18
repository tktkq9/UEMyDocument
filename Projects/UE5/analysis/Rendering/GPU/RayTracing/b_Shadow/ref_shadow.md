# GPU b Ref: Shadow シェーダーリファレンス

- シェーダー: `RayTracing/RayTracingOcclusionRGS.usf`, `RayTracingMaterialDefaultHitShaders.usf`
- CPU 対応: [[b_rt_shadow_ao]]
- 上位: [[01_raytracing_gpu_overview]]

---

## エントリポイント一覧

### RayTracingOcclusionRGS.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `Occlusion` | 654 | RGS / Inline CS | シャドウレイ発射・可視性評価 |

### RayTracingMaterialDefaultHitShaders.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `OpaqueShadowCHS` | 7 | CHS | 不透明シャドウ Hit（即遮蔽）|
| `HiddenMaterialCHS` | 14 | CHS | 非表示マテリアル Closest Hit |
| `HiddenMaterialAHS` | 20 | AHS | 非表示マテリアル Any Hit |

---

## Occlusion RGS/CS パラメータ

### 入力

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `TLAS` | `RaytracingAccelerationStructure` | シーン TLAS |
| `LightingChannelMask` | `uint` | ライティングチャンネルフィルタ |
| `SamplesPerPixel` | `uint` | ピクセルあたりサンプル数 |
| `NormalBias` | `float` | 自己交差防止オフセット |
| `LightScissor` | `int4` | ライトシザー矩形 |
| `PixelOffset` | `int2` | チェッカーボードオフセット |
| `TraceDistance` | `float` | 最大トレース距離 |
| `LODTransitionStart` / `End` | `float` | LOD 遷移範囲 |
| `bAcceptFirstHit` | `uint` | 最初のヒットで即採用（高速）|
| `bTransmissionSamplingDistanceCulling` | `uint` | 半透明距離カリング |
| `TransmissionSamplingTechnique` | `uint` | 半透明サンプリング手法 |
| `RejectionSamplingTrials` | `uint` | 拒否サンプリング試行数 |

### 出力

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `RWOcclusionMaskUAV` | `RWTexture2D<float4>` | 可視性マスク（デノイザー入力）|
| `RWRayDistanceUAV` | `RWTexture2D<float>` | ヒット距離 |
| `RWSubPixelOcclusionMaskUAV` | `RWTexture2D<float4>` | Hair サブピクセルマスク |

---

## パーミュテーション

| マクロ | 値 | 説明 |
|--------|-----|------|
| `COMPUTESHADER` | 0/1 | 0=RGS（HW RT） / 1=Inline CS |
| `USE_TRANSLUCENT_SHADOW` | 0/1 | 半透明シャドウ（AHS で透過率評価）|
| `HAIR_STRANDS_SHADOW` | 0/1 | Hair サブピクセルシャドウ |
| `SUBSTRATE_GBUFFER_FORMAT` | 0/1 | Substrate GBuffer 形式 |

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `RayTracingCommon.ush` | TraceOcclusionRay(), FDefaultPayload |
| `RayTracingDirectionalLight.ush` | Directional Light レイ生成 |
| `RayTracingRectLight.ush` | Rect Light サンプリング |
| `RayTracingSphereLight.ush` | Sphere Light サンプリング |
| `RayTracingCapsuleLight.ush` | Capsule Light サンプリング |
| `ScreenSpaceDenoise/SSDPublic.ush` | デノイザーインターフェイス |
| `TransmissionCommon.ush` | 半透明透過率計算 |
