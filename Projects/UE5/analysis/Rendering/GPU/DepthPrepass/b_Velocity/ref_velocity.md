# REF: Velocity Buffer シェーダー

- グループ: b - Velocity
- 詳細: [[detail_velocity]]
- CPU リファレンス: [[ref_velocity_rendering]]
- ソース: `Engine/Shaders/Private/VelocityShader.usf`

---

## MainVertexShader（VelocityShader.usf:52）

### エントリポイント

```hlsl
void MainVertexShader(
    FVertexFactoryInput Input,
    out FVelocityVSToPS Output
#if USE_GLOBAL_CLIP_PLANE
    , out float OutGlobalClipPlaneDistance : SV_ClipDistance
#endif
    )
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Vertex Shader |
| **目的** | 現フレーム・前フレームのクリップ座標を同時に計算して PS に渡す |

#### FVelocityInterpsVSToPS / FVelocityVSToPS

```hlsl
struct FVelocityInterpsVSToPS
{
    float4 PackedVelocityA : TEXCOORD6;  // 現フレームのクリップ座標（ステレオ時）
    float4 PackedVelocityC : TEXCOORD7;  // 前フレームのクリップ座標
    FVertexFactoryInterpolantsVSToPS FactoryInterpolants;
};

struct FVelocityVSToPS
{
    INVARIANT_OUTPUT float4 Position : SV_POSITION;  // 現フレームの位置
    FVelocityInterpsVSToPS Interps;
    FStereoVSOutput StereoOutput;
};
```

#### CPU バインド

```cpp
// VelocityRendering.cpp
// FOpaqueVelocityMeshProcessor::AddMeshBatch() でメッシュコマンドを構築
// DrawDynamicMeshCommands() → RHI DrawIndexedPrimitive
// RT: VelocityBuffer (R16G16_FLOAT)
// DepthTest: Equal（Depth Pre-pass の深度と一致するもののみ）
```

---

## MainPixelShader（VelocityShader.usf 後半）

### エントリポイント

```hlsl
void MainPixelShader(
    FVelocityInterpsVSToPS Inputs,
    in float4 SvPosition : SV_POSITION,
    out float4 OutColor  : SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Pixel Shader |
| **目的** | 前後フレームのスクリーン座標差分を VelocityBuffer に書き込む |
| **深度バイアス** | `GDepthBias = 0.001f`（z ファイティング防止）|

#### 出力フォーマット

```hlsl
// EncodeVelocityToTexture() の結果を R16G16F に書き込み
// Velocity.xy = ScreenPos - PrevScreenPos（-1〜1）
// エンコード後: 0.5 + Velocity * 0.5（0〜1）
```

---

## パーミュテーション

| マクロ | 説明 |
|-------|------|
| `TRANSLUCENCY_VELOCITY_FROM_DEPTH` | 半透明オブジェクトの Velocity を深度から計算 |
| `STEREO_MOTION_VECTORS` | ステレオレンダリング対応の Velocity |
| `VELOCITY_CLIPPED_DEPTH_PASS` | 半透明 Depth Clip Pass での Velocity |
| `USE_WORLD_POSITION_EXCLUDING_SHADER_OFFSETS` | WPO 除外ワールド座標を出力 |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.BasePassOutputsVelocity` | 0 | BasePass で直接 Velocity を出力（Depth Pre-pass 後でなく）|
| `r.VelocityOutputPass` | 0 | 0=EarlyZ後 / 1=BasePass後 |
| `r.MotionBlurSeparateCameraVelocity` | 0 | カメラ移動由来の Velocity を分離 |
