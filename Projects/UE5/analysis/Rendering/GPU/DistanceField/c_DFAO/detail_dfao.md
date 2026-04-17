# GPU c: DFAO — Global SDF コーントレース AO シェーダー

- シェーダー: `DistanceFieldObjectCulling.usf`, `DistanceFieldScreenGridLighting.usf`, `DistanceFieldLightingPost.usf`
- CPU 対応: [[c_dfao]] | [[ref_dfao]]
- 上位: [[01_distance_field_gpu_overview]]

---

## 概要

DFAO GPU パスは3つのシェーダーファイルに分かれた複数の Compute Shader で構成される。  
`GAODownsampleFactor = 2`（半解像度）で処理し、最後にアップスケールする。

---

## パス構成

```
1. CullObjectsForViewCS（DistanceFieldObjectCulling.usf）
   → フラスタム内の SDF オブジェクトをカリング

2. ComputeDistanceFieldNormalCS（DistanceFieldScreenGridLighting.usf）
   → Downsampled GBuffer から法線を計算（コーントレースの起点方向）

3. BuildTileObjectListCS（DistanceFieldObjectCulling.usf）
   → タイル（32x32）ごとに影響オブジェクトリストを構築

4. AOLevelSetCS（DistanceFieldScreenGridLighting.usf）
   → コーントレース: Global SDF をサンプリングして遮蔽を積分

5. UpsampleBentNormalAO（DistanceFieldLightingPost.usf）
   → 半解像度 BentNormal を全解像度にアップスケール
```

---

## CullObjectsForViewCS

`DistanceFieldObjectCulling.usf — CullObjectsForViewCS`

```hlsl
[numthreads(UPDATEOBJECTS_THREADGROUP_SIZE, 1, 1)]
void CullObjectsForViewCS(...)
{
    uint ObjectIndex = DispatchThreadId;

    FDFObjectBounds DFObjectBounds = LoadDFObjectBounds(ObjectIndex);
    float3 TranslatedCenter = DFFastToTranslatedWorld(DFObjectBounds.Center, ...);

    // ビュー距離 + フラスタム交差テスト
    if (DistanceToViewSq < Square(AOMaxViewDistance + SphereRadius)
        && ViewFrustumIntersectSphere(TranslatedCenter, SphereRadius + AOObjectMaxDistance))
    {
        // 描画距離チェック後、出力バッファへ追加
        InterlockedAdd(RWObjectIndirectArguments[1], ...);
    }
}
```

フラスタムカリング + 距離カリングを組み合わせ、通過したオブジェクトを  
`RWCulledObjectData` バッファへ Scatter-Append する。

---

## ComputeDistanceFieldNormalCS

`DistanceFieldScreenGridLighting.usf`

```hlsl
[numthreads(THREADGROUP_SIZEX, THREADGROUP_SIZEY, 1)]
void ComputeDistanceFieldNormalCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    // ピクセル座標 (GAODownsampleFactor 分オフセット)
    float2 PixelCoord = DispatchThreadId.xy * DOWNSAMPLE_FACTOR + ...;
    
    float SceneDepth = CalcSceneDepth(ScreenUV);
    float3 WorldNormal = GBuffer.WorldNormal;  // GBuffer から読み取り
    
    // 深度+法線をエンコードして出力
    RWDistanceFieldNormal[OutputPos] = EncodeDownsampledGBuffer(WorldNormal, SceneDepth);
}
```

Substrate GBuffer 形式にも対応（`#if SUBSTRATE_GBUFFER_FORMAT==1`）。

---

## コーントレース（AOLevelSetCS）

`DistanceFieldScreenGridLighting.usf`

DFAO の中核処理。Global SDF を複数方向へコーントレースして遮蔽を積分する。

```hlsl
// 擬似コード
void AOLevelSetCS(...)
{
    // 1. このピクセルの位置・法線を取得（DistanceFieldNormalTexture から）
    float3 WorldNormal = DecodeNormal(DistanceFieldNormal[PixelCoord]);
    
    // 2. 複数コーン方向（GetSpacedVectors() でフレームごとに回転）を取得
    TArray<float3> ConeDirections = GetSpacedVectors(FrameNumber, WorldNormal);
    
    // 3. 各コーン方向でトレース
    for each ConeDir in ConeDirections:
    {
        float ConeVisibility = 1.0f;
        float StepDist = AOGlobalDFStartDistance;
        
        while (StepDist < AOGlobalMaxOcclusionDistance)
        {
            // Global SDF をサンプリング（クリップマップ選択）
            float Distance = SampleGlobalDistanceField(WorldPos + ConeDir * StepDist);
            
            // コーン開口角に対して遮蔽率を計算
            float SphereRadius = StepDist * TanConeAngle;
            ConeVisibility = min(ConeVisibility, Distance / SphereRadius);
            
            // ステップサイズをジオメトリカルに増加
            StepDist += max(Distance, MinStepDistance) * StepScale;
        }
        Occlusion += 1 - saturate(ConeVisibility);
    }
    
    // 4. BentNormal と AO をエンコードして出力
    BentNormal = WeightedSum(ConeDirections, Visibilities);
    AO = Occlusion / NumCones;
}
```

---

## r.AOQuality によるパーミュテーション

| `r.AOQuality` | コーン数 | ステップ数 | 品質 |
|-------------|---------|---------|------|
| 1（低） | 4 | 少ない | 高速・低品質 |
| 2（中） | 6 | 中程度 | バランス |
| 3（高） | 9 | 多い | 低速・高品質 |

HLSL 内でシェーダー `#define` マクロにより静的分岐。

---

## UpsampleBentNormalAO

`DistanceFieldLightingPost.usf`

半解像度（1/2）の BentNormal/AO テクスチャを全解像度に復元する。

```hlsl
void UpsampleBentNormalAO(...)
{
    // バイラテラルアップスケール（深度差で重み付け）
    // 半解像度から4近傍をサンプリング → 深度の近いサンプルを優先
    float3 BentNormal = BilateralUpsample(
        BentNormalAOTexture, FullResDepth, HalfResDepth);
    
    // AO フェードアウト（遠距離ではフェード）
    float Fade = saturate(1.0f - (Depth - AOMaxViewDistance + FadeRange) / FadeRange);
    BentNormal = lerp(float3(0, 0, 1), BentNormal, Fade);
}
```

---

## Temporal 統合

DFAO は `GetSpacedVectors(FrameNumber)` でフレームごとに異なるコーン方向を使用し、  
TAA（Temporal Anti-Aliasing）のアキュムレーションで高品質な AO を擬似的に得る。

- フレームN: コーン方向セット A（4方向）
- フレームN+1: コーン方向セット B（4方向・セットAと分散して配置）
- TAA が時間方向で統合 → 実効的に多コーン数と同等の品質を実現

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `DistanceFieldLightingShared.ush` | Object SDF バッファアクセス・共通型 |
| `DistanceFieldAOShared.ush` | AO 専用定数・エンコード関数 |
| `DistanceField/GlobalDistanceFieldShared.ush` | Global SDF クリップマップサンプリング |
