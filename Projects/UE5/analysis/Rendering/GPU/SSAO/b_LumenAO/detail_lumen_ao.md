# Lumen Screen Space Short Range AO シェーダー詳細

- グループ: b - Lumen AO
- GPU 概要: [[01_ssao_gpu_overview]]
- CPU 詳細: [[a_ssao]]
- リファレンス: [[ref_lumen_ao]]

---

## 概要

Lumen 有効時に使われる **Screen Space Short Range AO**。  
通常の SSAO/GTAO の代わりに、Lumen の GI パイプラインと統合されたより短距離の AO を計算する。  
Bent Normal（遮蔽方向を考慮した法線）も同時に計算し、Lumen の Diffuse GI Composite で活用される。

---

## レンダリングパスの構成

```
LumenScreenSpaceBentNormal パス
  │
  ├─ 入力: SceneDepth, GBuffer Normal, SceneColor
  ├─ CS: ScreenSpaceShortRangeAOCS()
  │   ├─ 各ピクセルから短距離のスクリーンスペースレイを飛ばす
  │   ├─ 遮蔽チェック（SceneDepth と比較）
  │   └─ AO 値 + Bent Normal を出力
  └─ 出力: Bent Normal テクスチャ（Lumen DiffuseIndirect Composite で使用）
```

---

## 入出力

### 入力

| リソース | 説明 |
|---------|------|
| `SceneDepth` | シーン深度 |
| `GBuffer Normal` | ワールド法線（GBufferA から）|
| `LumenScreenSpaceBentNormal` パラメータ群 | サンプル数・半径等 |

### 出力

| リソース | フォーマット | 内容 |
|---------|------------|------|
| Bent Normal テクスチャ | `R8G8B8A8` | xyz = Bent Normal 方向、w = AO 値 |

---

## シェーダーコアロジック（LumenScreenSpaceBentNormal.usf）

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ScreenSpaceShortRangeAOCS(uint2 PixelPos : SV_DispatchThreadID)
{
    float SceneDepth = ...;
    float3 WorldNormal = DecodeGBufferNormal(GBufferA, UV);
    float3 TranslatedWorldPos = ReconstructTranslatedWorldPosition(UV, SceneDepth);

    float3 BentNormal = float3(0, 0, 0);
    float AO = 0;

    // 複数方向のスクリーンスペースレイをサンプル
    for (int SampleIndex = 0; SampleIndex < NUM_SAMPLES; SampleIndex++)
    {
        float2 SampleOffset = GetSampleOffset(SampleIndex, PixelRandomAngle);
        float2 SampleUV = UV + SampleOffset * AO_RADIUS_PIXELS;

        float SampleDepth = SceneDepth.SampleLevel(SampleUV, 0);
        float3 SamplePos = ReconstructTranslatedWorldPosition(SampleUV, SampleDepth);

        float3 Direction = SamplePos - TranslatedWorldPos;
        float Distance = length(Direction);

        // 法線方向との角度で遮蔽を判定
        float NdotDir = dot(WorldNormal, normalize(Direction));
        if (NdotDir > 0 && Distance < MaxAORadius)
        {
            AO += NdotDir * OcclusionWeight(Distance);
            BentNormal += Direction * OcclusionWeight(Distance);
        }
    }

    // 正規化して出力
    AO = saturate(1.0 - AO / NUM_SAMPLES);
    BentNormal = length(BentNormal) > 0 ? normalize(BentNormal) : WorldNormal;

    RWBentNormalTexture[PixelPos] = float4(BentNormal * 0.5 + 0.5, AO);
}
```

---

## CPU 呼び出しの流れ

```
RenderLumenScreenSpaceBentNormal()         // IndirectLightRendering.cpp
  │
  ├─ 有効条件: r.Lumen.ScreenBentNormal=1 かつ Lumen GI 有効
  │
  ├─ AddPass(RDG_EVENT_NAME("LumenScreenSpaceShortRangeAO"))
  │   CS: ScreenSpaceShortRangeAOCS()
  │   GroupCount: View.ViewRect / THREADGROUP_SIZE
  │
  └─ 出力 BentNormal テクスチャ
       → RenderDiffuseIndirectAndAmbientOcclusion() で使用
       → DiffuseIndirectComposite.usf に渡される
```

---

## Lumen GI との連携

```
Lumen Diffuse GI パイプライン:
  [1] Screen Probe Gather → Diffuse GI バッファ
  [2] ScreenSpaceShortRangeAOCS → Bent Normal テクスチャ ← ここ
  [3] DiffuseIndirectComposite.usf#MainPS()
        DiffuseGI × (1 - AO) + Specular → SceneColor
        BentNormal → Screen Probe のサンプル方向補正
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `Lumen/LumenScreenSpaceBentNormal.usf` | CS | Short Range AO + Bent Normal 本体 |
| `DiffuseIndirectComposite.usf` | PS | Lumen GI + AO の最終合成（Bent Normal を受け取る）|
| `PostProcessAmbientOcclusion.usf` | PS/CS | 非 Lumen 時の SSAO/GTAO（Lumen 有効時はスキップ）|
