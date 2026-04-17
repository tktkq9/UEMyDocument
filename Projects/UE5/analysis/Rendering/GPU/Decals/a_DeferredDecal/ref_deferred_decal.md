# REF: Deferred Decal シェーダー

- グループ: a - DeferredDecal
- 詳細: [[detail_deferred_decal]]
- CPU リファレンス: [[ref_deferred_decal]]
- ソース: `Engine/Shaders/Private/DeferredDecal.usf`  
          `Engine/Shaders/Private/DecalCommon.ush`

---

## MainVS（DeferredDecal.usf:62）

### エントリポイント

```hlsl
void MainVS(
    in FStereoVSInput StereoInput,
    in float4 InPosition : ATTRIBUTE0,
    out float4 OutPosition : SV_POSITION,
    out FStereoVSOutput StereoOutput)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Vertex Shader |
| **目的** | デカールボックス（単位立方体）を FrustumComponentToClip でクリップ空間に変換 |
| **ユニフォーム** | `FrustumComponentToClip`（デカールコンポーネント → クリップ空間の変換行列）|

#### CPU バインド

```cpp
// DeferredDecalRendering.cpp
FDeferredDecalVS::FParameters* PassParameters = ...;
PassParameters->DecalPositionHigh = FVector4f(Decal.ComponentTranslatedWorldTransform.GetOrigin(), 0);
PassParameters->FrustumComponentToClip = FMatrix44f(FrustumToClip);
PassParameters->SvPositionToDecal = FMatrix44f(SvPositionToDecal);
PassParameters->DecalToWorld = FMatrix44f(DecalToWorld);
```

---

## FPixelShaderInOut_MainPS（DeferredDecal.usf:125/315）

### エントリポイント（GBuffer / DBuffer 共通）

```hlsl
void FPixelShaderInOut_MainPS(
    inout FPixelShaderIn In,
    inout FPixelShaderOut Out,
    uint ArrayIndex)  // Stereo Array Index
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Pixel Shader |
| **目的** | SceneDepth からワールド座標を復元してデカール空間に変換し、マテリアルを評価して書き込む |
| **マクロ** | `DECAL_RENDERTARGETMODE`（DBuffer/GBuffer 書き込み先）/ `DECAL_RENDERSTAGE` / `IS_DECAL` |

#### パーミュテーション

| マクロ | 値 | 説明 |
|-------|------|------|
| `DECAL_RENDERTARGETMODE` | 0=DBuffer / 1=GBuffer / 2=SceneColor | 出力先 |
| `DECAL_RENDERSTAGE` | 0=BeforeBasePass / 1=BeforeLighting / 2=Emissive | 実行タイミング |
| `DECAL_BLENDMODE` | 0〜4 | Translucent/Stain/Normal/Emissive/AlphaComposite |
| `SUBSTRATE_ENABLED` | 0/1 | Substrate マテリアルシステム対応 |

#### 主要ユニフォーム

| バインド名 | 型 | 説明 |
|-----------|------|------|
| `SvPositionToDecal` | `float4x4` | SV_Position → デカールローカル空間 |
| `DecalToWorld` | `float4x4` | デカール → ワールド空間（法線変換用）|
| `DecalToWorldInvScale` | `float3` | スケール逆数（法線正規化用）|
| `DecalParams` | `float2` | x=サイズフェード α, y=不透明度 |
| `DecalColorParam` | `float4` | デカールカラー（Material.DecalColor）|

---

## DecalCommon.ush 主要関数

### FDecalData 構造体

```hlsl
struct FDecalData
{
    float3 LocalPosition;  // デカールローカル座標（[-1,1]）
    float  FadeAlpha;      // フェードアルファ
    float3 WorldNormal;    // マテリアル法線（ワールド空間）
    float3 DiffuseColor;   // マテリアル BaseColor
    float  Roughness;
    float  Metallic;
    float  Opacity;
};
```

### WriteDBufferData

```hlsl
void WriteDBufferData(inout FPixelShaderOut Out, FDecalData DecalData, ...)
// DBufferA: BaseColor × Opacity
// DBufferB: Normal × Opacity（lerp 係数も同時に出力）
// DBufferC: Roughness/Metallic × Opacity
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.DBuffer` | 1 | DBuffer デカール有効 |
| `r.Decal.StencilSizeThreshold` | 0.1 | ステンシル最適化閾値 |
| `r.Decal.ForceFullDepthPass` | 0 | 深度パス強制 |
