# SSAO / GTAO シェーダー詳細

- グループ: a - SSAO
- GPU 概要: [[01_ssao_gpu_overview]]
- CPU 詳細: [[a_ssao]]
- リファレンス: [[ref_ssao]]

---

## 概要

UE5 の Screen Space AO は **GTAO（Ground-Truth Ambient Occlusion）** を中心に構成される。  
複数のパーミュテーション（Setup → HorizonSearch → Integrate → SpatialFilter → TemporalFilter → Upsample）を  
シーン条件（解像度・品質設定・TAA）に応じて組み合わせて実行する。

---

## レンダリングパスの構成

```
[1] Setup Pass（半解像度縮小）
    PS: MainSetupPS()
    入力: SceneDepth + GBuffer Normal
    出力: 半解像度 Depth+Normal テクスチャ

[2] Horizon Search（GTAO）
    CS: HorizonSearchCS()  or  PS: HorizonSearchPS()
    入力: 半解像度 Depth+Normal
    出力: HorizonAngle テクスチャ（各方向のHorizon Angle）

[3] Inner Integrate（GTAO）
    CS: GTAOInnerIntegrateCS()  or  PS: GTAOInnerIntegratePS()
    入力: HorizonAngle テクスチャ
    出力: AO + Bent Normal（R8 または R8G8）

    ※ EAsyncCombinedSpatial モードでは [2]+[3] が GTAOCombinedCS() に統合

[4] Spatial Filter
    CS: GTAOSpatialFilterCS()  or  PS: GTAOSpatialFilterPS()
    入力: Raw AO テクスチャ
    出力: スペーシャルフィルタ済み AO

[5] Temporal Filter
    CS: GTAOTemporalFilterCS()  or  PS: GTAOTemporalFilterPS()
    入力: 現フレーム AO + 前フレーム History
    出力: テンポラル蓄積 AO

[6] Upsample（半解像度 → 全解像度）
    PS: GTAOUpsamplePS()
    入力: テンポラル AO（半解像度）+ SceneDepth（全解像度）
    出力: 全解像度 AO テクスチャ
```

---

## 入出力

### 入力（Horizon Search）

| リソース | 説明 |
|---------|------|
| `SceneDepth` | シーン深度（全解像度 or 半解像度）|
| `GBufferA`（Normal）| ワールド法線 |
| `GTAO_TemporalParameters` | ジッタ・テンポラル設定 |

### 出力

| リソース | フォーマット | 内容 |
|---------|------------|------|
| AO テクスチャ | R8 | AO 値（0=完全遮蔽 / 1=開放）|
| AO + Bent Normal | R8G8 | AO + Bent Normal（x,y）（高品質時）|

---

## シェーダーコアロジック

### GTAO Horizon Search

```hlsl
// HorizonSearchCS() / HorizonSearchPS()
// 各ピクセルから SAMPLE_STEPS × NUM_DIRECTIONS 方向に Ray を飛ばして
// Horizon Angle（HorizonAngle1, HorizonAngle2）を計算

for (int i = 0; i < NUM_DIRECTIONS; i++)
{
    float2 Direction = GetSampleDirection(i, PixelRandomAngle);
    float HorizonAngle = FindHorizonAngle(PixelPos, Direction, SceneDepth, Radius);
    // HorizonAngle → GTAOInnerIntegrate で積分
}
```

### GTAO Inner Integrate

```hlsl
// GTAOInnerIntegrateCS() / GTAOInnerIntegratePS()
// Horizon Angle から AO値を積分（Alchemy AO 近似）

// OPTIMIZATION_O1=1: 正規化なし・簡略式（Alchemy-like）
// OPTIMIZATION_O1=0: 重み付き平均（標準 GTAO）
float AO = IntegrateHorizonAngle(HorizonAngle1, HorizonAngle2, Normal);
```

### GTAO Spatial Filter

```hlsl
// GTAOSpatialFilterCS()
// 法線・深度aware なクロスバイラテラルフィルタ
// QUAD_MESSAGE_PASSING_BLUR = 2: 法線考慮
// QUAD_MESSAGE_PASSING_BLUR = 3: 法線+深度考慮
float Weight = GetBilateralWeight(CenterDepth, SampleDepth, CenterNormal, SampleNormal);
AO = saturate(AO + SampleAO * Weight) / TotalWeight;
```

### GTAO Temporal Filter

```hlsl
// GTAOTemporalFilterCS()
// 前フレームの AO を再投影して現フレームとブレンド
float2 Velocity = Texture2DSampleLevel(VelocityTexture, UV, 0).xy;
float HistoryAO = Texture2DSampleLevel(HistoryBuffer, UV - Velocity, 0).r;
float BlendFactor = TemporalWeight;  // 通常 0.1〜0.9
float FinalAO = lerp(CurrentAO, HistoryAO, BlendFactor);
```

### パーミュテーション（SHADER_QUALITY）

| SHADER_QUALITY | SAMPLE_STEPS | USE_SAMPLESET | BLUR |
|---------------|------------|-------------|------|
| 0 (最低) | 1 | 1 | 0 |
| 1 (低) | 1 | 1 | 2 |
| 2 (中) | 2 | 1 | 2 |
| 3 (高) | 3 | 1 | 0 |
| 4 (最高) | 3 | 3 | 0 |

---

## CPU 呼び出しの流れ

```
CompositionLighting.ProcessAfterOcclusion()   // CompositionLighting.cpp
  │
  ├─ FSSAOHelper::IsAmbientOcclusionAsyncCompute() で AsyncCompute 判定
  │
  ├─ [GTAO モード]
  │   RenderGTAO(GraphBuilder, View, SceneTextures, ...)
  │     ├─ FGTAOContext 初期化（EGTAOType で分岐）
  │     ├─ AddPass(HorizonSearchCS/PS)
  │     ├─ AddPass(GTAOInnerIntegrateCS) または AddPass(GTAOCombinedCS)
  │     ├─ AddPass(GTAOSpatialFilterCS)
  │     ├─ AddPass(GTAOTemporalFilterCS)
  │     └─ AddPass(GTAOUpsamplePS)
  │
  └─ [従来 SSAO モード（r.AmbientOcclusion.Method=0）]
      FRCPassPostProcessAmbientOcclusion::Process()
        ├─ AddPass(MainSetupPS)
        ├─ AddPass(MainCS / MainPS)
        └─ AddPass(MainSSAOSmoothCS)
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `PostProcessAmbientOcclusion.usf` | PS/CS | GTAO・SSAO 全パスの本体 |
| `PostProcessAmbientOcclusionMobile.usf` | PS | モバイル向け簡略 SSAO |
| `ScreenPass.ush` | ヘッダ | スクリーンスペースパス共通ユーティリティ |
| `DeferredShadingCommon.ush` | ヘッダ | GBuffer デコード |
