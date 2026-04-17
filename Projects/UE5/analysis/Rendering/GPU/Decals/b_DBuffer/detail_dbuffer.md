# DBuffer デカール シェーダー詳細

- グループ: b - DBuffer
- GPU 概要: [[01_decals_gpu_overview]]
- CPU 詳細: [[b_dbuffer]]
- リファレンス: [[ref_dbuffer]]

---

## 概要

**DBuffer** は BasePass の前に実行される Pre-GBuffer デカール合成方式。  
デカールマテリアルの BaseColor/Normal/Roughness/Metallic を3枚の DBuffer テクスチャに書き込み、  
BasePass シェーダー内で GBuffer 書き込み直前に `ApplyDBufferData()` で合算する。  
ライティング計算に完全に統合されるため、ライティングと完全整合したデカールが実現できる。

---

## レンダリングパスの構成

```
DBuffer Pass（EDecalRenderStage::BeforeBasePass）
  │
  ├─ VS: DeferredDecal.usf#MainVS()（Deferred Decal と同じ VS）
  └─ PS: DeferredDecal.usf#FPixelShaderInOut_MainPS()
          DECAL_RENDERTARGETMODE = DECAL_RENDERTARGETMODE_DBUFFER
          → WriteDBufferData() で DBufferA/B/C に Lerp 係数付きで書き込み

BasePass 内（各メッシュのマテリアルシェーダー）:
  → ApplyDBufferData(DBufferA, DBufferB, DBufferC)
     BaseColor   = lerp(BaseColor, DBufferA.rgb, DBufferA.a)
     Normal      = lerp(Normal,    DBufferB.rgb, DBufferB.a)
     Roughness   = lerp(Roughness, DBufferC.r,   DBufferC.b)
     Metallic    = lerp(Metallic,  DBufferC.g,   DBufferC.b)
```

---

## DBuffer テクスチャ仕様

| テクスチャ | フォーマット | チャンネル |
|-----------|------------|----------|
| DBufferA | `R8G8B8A8` | rgb = BaseColor, a = Lerp 係数（Opacity）|
| DBufferB | `R8G8B8A8` | rgb = Normal（Octahedron エンコード等）, a = Lerp 係数 |
| DBufferC | `R8G8B8A8` | r = Roughness, g = Metallic, b = Lerp 係数 |

---

## シェーダーコアロジック

### DBuffer 書き込み（DeferredDecal.usf）

```hlsl
// FPixelShaderInOut_MainPS 内（DECAL_RENDERTARGETMODE_DBUFFER 時）
void FPixelShaderInOut_MainPS(inout FPixelShaderIn In, inout FPixelShaderOut Out, uint ArrayIndex)
{
    // デカール空間への変換・マテリアル評価（Deferred Decal と共通）
    float3 DecalVector = ComputeDecalVector(In.SvPosition);
    clip(1 - abs(DecalVector));

    // マテリアル評価
    FDecalData DecalData = EvaluateDecalMaterial(DecalVector);

    // DBuffer に書き込み（Lerp 係数を Alpha チャンネルに格納）
    Out.MRT[0] = float4(DecalData.DiffuseColor, DecalData.Opacity);  // DBufferA
    Out.MRT[1] = float4(EncodeNormal(DecalData.WorldNormal), DecalData.Opacity);  // DBufferB
    Out.MRT[2] = float4(DecalData.Roughness, DecalData.Metallic, 0, DecalData.Opacity);  // DBufferC
}
```

### BasePass での DBuffer 適用（DBufferCommon.ush）

```hlsl
// ApplyDBufferData() - BasePass 内で GBuffer 書き込み直前に呼ばれる
void ApplyDBufferData(
    FDBufferData DBufferData,
    inout half3 WorldNormal,
    inout half3 SubsurfaceColor,
    inout half Roughness,
    inout half3 BaseColor,
    inout half Metallic,
    inout half Specular)
{
    // DBufferA.a = Lerp 係数（0=デカールなし / 1=デカール全面）
    BaseColor = lerp(BaseColor, DBufferData.ColorOpacity.rgb, DBufferData.ColorOpacity.a);
    WorldNormal = lerp(WorldNormal, DBufferData.NormalOpacity.xyz, DBufferData.NormalOpacity.a);
    Roughness = lerp(Roughness, DBufferData.RoughnessMetallicSpecular.r, DBufferData.RoughnessMetallicSpecularOpacity);
    Metallic  = lerp(Metallic,  DBufferData.RoughnessMetallicSpecular.g, DBufferData.RoughnessMetallicSpecularOpacity);
}
```

---

## CPU 呼び出しの流れ

```
RenderDeferredDecals()                        // DeferredDecalRendering.cpp
  │
  ├─ [EDecalRenderStage::BeforeBasePass]
  │   AllocateDBufferTextures() → DBufferA/B/C（R8G8B8A8）を RDG で確保
  │   ClearDBufferTextures() → 全ピクセルを 0 にクリア（Lerp係数 0 = デカールなし）
  │
  ├─ デカールソート（奥 → 手前の順）
  ├─ AddPass(RDG_EVENT_NAME("DeferredDecals DBuffer"))
  │   DepthTest: GreaterEqual（SceneDepth で front-face のみ）
  │   DepthWrite: false
  │   BlendState: Additive for Lerp 係数 / Alpha-Blend for Color
  │
  └─ BasePass から FDBufferTextures をバインド
      → BasePassPixelShader.usf 内で ApplyDBufferData() を呼び出し
```

---

## DBuffer vs 後置デカールの比較

| 特性 | DBuffer | 後置デカール |
|-----|---------|-----------|
| 実行タイミング | BasePass 前 | GBuffer 後 |
| ライティング統合 | 完全 | 制限あり |
| Static Lighting | 対応 | 非対応 |
| コスト | GBuffer メモリ + 3RT | GBuffer 上書きのみ |

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `DeferredDecal.usf` | VS/PS | DBuffer 書き込み本体（同ファイルをパーミュテーションで共有）|
| `DBufferCommon.ush` | ヘッダ | `ApplyDBufferData()` / `FDBufferData` 構造体 |
| `Substrate/SubstrateDBuffer.usf` | CS | Substrate マテリアルの DBuffer 処理 |
