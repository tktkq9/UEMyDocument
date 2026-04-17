# Deferred Decal シェーダー詳細

- グループ: a - DeferredDecal
- GPU 概要: [[01_decals_gpu_overview]]
- CPU 詳細: [[a_deferred_decal]]
- リファレンス: [[ref_deferred_decal]]

---

## 概要

**Deferred Decal** はデカールボックスをラスタライズして SceneDepth からワールド座標を復元し、  
デカール空間に変換してマテリアルを評価する。BlendMode に応じて GBuffer を直接上書きする。  
DBuffer（Pre-GBuffer）と GBuffer 後置の2系統があり、同じシェーダーをパーミュテーションで切り替える。

---

## レンダリングパスの構成

```
Deferred Decal パス
  │
  ├─ VS: DeferredDecal.usf#MainVS()
  │   デカールボックス頂点 → FrustumComponentToClip で変換
  │   → SV_POSITION（深度テストで GBuffer に一致するピクセルのみ処理）
  │
  └─ PS: DeferredDecal.usf#FPixelShaderInOut_MainPS()
      1. SvPosition.z を SceneDepth に差し替え（デカールの depth = シーン depth）
      2. SvPositionToDecal 行列でデカール空間に変換（[-1,1] ボックス）
      3. ボックス外のピクセルを clip()
      4. デカール空間 UV → マテリアル評価
      5. FDecalData を構築して各レンダーターゲットに書き込み
```

---

## 入出力

### 入力

| リソース | 説明 |
|---------|------|
| `SceneDepthTexture` | GBuffer の SceneDepth（ワールド座標復元用）|
| `SvPositionToDecal` | SV_Position → デカールローカル空間の変換行列 |
| `FrustumComponentToClip` | デカールボックス → クリップ空間 |
| `DecalToWorld` | デカール → ワールド空間（法線変換用）|

### 出力（BlendMode 別）

| BlendMode | 出力先 | 内容 |
|-----------|--------|------|
| DBuffer | DBufferA/B/C | BaseColor / Normal / Roughness（Lerp 係数付き）|
| Translucent | GBufferA/B/C | GBuffer を直接上書き |
| Normal Only | GBufferB | 法線のみ上書き |
| Emissive | SceneColor | 加算合成 |
| AO | GBufferE | AO 値 |

---

## シェーダーコアロジック

### MainVS

```hlsl
void MainVS(
    in float4 InPosition : ATTRIBUTE0,
    out float4 OutPosition : SV_POSITION,
    ...)
{
    // FrustumComponentToClip: デカールボックス → クリップ空間
    OutPosition = mul(InPosition, FrustumComponentToClip);
}
```

### FPixelShaderInOut_MainPS（DBuffer / GBuffer 共通）

```hlsl
void FPixelShaderInOut_MainPS(inout FPixelShaderIn In, inout FPixelShaderOut Out, uint ArrayIndex)
{
    float2 ScreenUV = SvPositionToBufferUV(In.SvPosition);

    // 1. SvPosition.z を SceneDepth に差し替え
    In.SvPosition.z = LookupDeviceZ(ScreenUV);
    In.SvPosition.w = ConvertFromDeviceZ(In.SvPosition.z);

    // 2. デカール空間に変換（DecalVector = [-1,1] ボックス内座標）
    float4 DecalVectorHom = mul(float4(In.SvPosition.xyz, 1), SvPositionToDecal);
    float3 DecalVector = DecalVectorHom.xyz / DecalVectorHom.w;

    // 3. ボックス外クリップ
    clip(1 - abs(DecalVector));

    // 4. デカール UV = DecalVector.xy * 0.5 + 0.5
    float2 DecalUV = DecalVector.xy * float2(0.5, -0.5) + 0.5;

    // 5. マテリアル評価
    FMaterialPixelParameters MaterialParameters = GetMaterialPixelParameters_Decal(DecalUV, ...);
    // → デカールマテリアルの BaseColor, Normal, Roughness を取得

    // 6. BlendMode に応じてレンダーターゲットに書き込み
    WriteDBufferData(Out, FDecalData, ...);  // or WriteGBufferData(Out, ...)
}
```

---

## CPU 呼び出しの流れ

```
RenderDeferredDecals()                        // DeferredDecalRendering.cpp
  │
  ├─ FDecalVisibilityTaskData から可視デカールリストを取得
  ├─ FDecalBlendDesc でレンダーステージ・BlendMode を判定
  │
  ├─ [EDecalRenderStage::BeforeBasePass]
  │   → DBuffer パス（Pre-GBuffer）
  │
  ├─ [EDecalRenderStage::BeforeLighting]
  │   → GBuffer デカール（Translucent/Stain/Normal）
  │
  └─ [EDecalRenderStage::Emissive]
      → SceneColor への加算合成

  各ステージで:
    AddPass(RDG_EVENT_NAME("DeferredDecal"))
    VS: MainVS + PS: FPixelShaderInOut_MainPS
    BlendState = BlendMode 依存（lerp / additive 等）
    DepthTest: Equal（SceneDepth と一致するのみ）
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `DeferredDecal.usf` | VS/PS | デカール投影・マテリアル評価・書き込み本体 |
| `DecalCommon.ush` | ヘッダ | FDecalData / WriteDBufferData / DecalBlend ユーティリティ |
| `Substrate/SubstrateDBuffer.ush` | ヘッダ | Substrate マテリアルの DBuffer 対応 |
