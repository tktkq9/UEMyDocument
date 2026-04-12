# VSM Projection シェーダー詳細

- グループ: d - Projection
- GPU 概要: [[01_vsm_gpu_overview]]
- CPU 詳細: [[d_vsm_projection]]
- リファレンス: [[ref_projection]]

---

## 概要

VSM の投影パイプライン。GBuffer のピクセルが影に入っているかを判定し、  
Shadow Mask テクスチャに書き込む。  
**SMRT（Shadow Map Ray Tracing）** でソフトシャドウ（半影）を推定する高品質な実装。

```
VirtualShadowMapProjection CS（VirtualShadowMapProjection.usf）
  → SceneDepth から TranslatedWorldPosition を復元
  → ONE_PASS_PROJECTION: 全ローカルライトを1 Dispatch で処理（OutShadowMaskBits）
  → Per-Light（通常）: 1ライト1 Dispatch（OutShadowFactor）

VirtualShadowMapProjectionComposite PS（VirtualShadowMapProjectionComposite.usf）
  → ShadowFactor / ShadowMaskBits を Shadow Attenuation テクスチャに合成
```

---

## VirtualShadowMapProjection CS のコアロジック

```hlsl
[numthreads(WORK_TILE_SIZE, WORK_TILE_SIZE, 1)]
void VirtualShadowMapProjection(
    uint3 GroupId       : SV_GroupID,
    uint  GroupIndex    : SV_GroupIndex,
    uint3 DispatchThreadId : SV_DispatchThreadID
)
```

`WORK_TILE_SIZE` × `WORK_TILE_SIZE` タイルを Morton Order でアクセス（ページキャッシュ効率向上）。

### One-Pass Projection（ONE_PASS_PROJECTION=1）

複数のローカルライトを1 Dispatch で処理する高効率モード。

```hlsl
#if ONE_PASS_PROJECTION
    const FCulledLightsGridHeader LightGridHeader =
        VirtualShadowMapGetLightsGridHeader(LocalPixelPos, SceneDepth);

    uint4 ShadowMask = InitializePackedShadowMask();

    for (uint Index = 0; Index < LightGridHeader.NumLights; ++Index)
    {
        FLocalLightData LightData = VirtualShadowMapGetLocalLightData(LightGridHeader, Index);

        // MegaLights はソート末尾にある → ここで終了
        if (UnpackIsHandledByMegaLights(LightData)) break;

        int VirtualShadowMapId = LightData.Internal.VirtualShadowMapId;
        FVirtualShadowMapSampleResult Result = ProjectLight(
            VirtualShadowMapId, Light, ShadingInfo,
            PixelPos, SceneDepth, ScreenRayLengthWorld,
            TranslatedWorldPosition, Noise, SubsurfaceOpacityMFP);

        FilterVirtualShadowMapSampleResult(PixelPos, Result);
        PackShadowMask(ShadowMask, Result.ShadowFactor, Index);
    }

    OutShadowMaskBits[PixelPos] = ShadowMask;  // uint4: 最大 16 ライト分
#endif
```

### Per-Light Projection（ONE_PASS_PROJECTION=0）

単一ライト（ディレクショナルライトや重要ライト）を独立した Dispatch で処理。

```hlsl
FVirtualShadowMapSampleResult Result = ProjectLight(
    LightUniformVirtualShadowMapId, Light, ShadingInfo, ...);

OutShadowFactor[PixelPos] = Result.ShadowFactor.xx;
// .x = 影の強度（0=完全な影, 1=完全に明るい）
// .y = サブサーフェス散乱用の影強度（別チャンネル）
```

### ProjectLight() — SMRT の内部処理

`VirtualShadowMapProjection.usf` 内の中核関数。  
SMRT（Shadow Map Ray Tracing）でソフトシャドウを実現する。

```
1. TranslatedWorldPosition → ライトの射影空間へ変換
2. Virtual Page テーブルを参照して対応する物理ページを取得
3. SMRT ステップ（VirtualShadowMapSMRTTemplate.ush）:
   a. ライト方向に沿って複数のサンプルを取得
   b. 各サンプルで shadow depth と比較（遮蔽判定）
   c. ExtrapolateMaxSlope() で半影幅を推定
4. Blue Noise ジッターで時間的安定性を確保
5. FVirtualShadowMapSampleResult を返す
   (.ShadowFactor: 0=完全影, 1=完全明)
```

### サブサーフェス散乱対応

```hlsl
// SubsurfaceOpacityMFP が有効 → ライトの SourceRadius を拡大（半影を広げる）
if (SubsurfaceOpacityMFP.Data < 1.0f)
    Light.SourceRadius = max(Light.SourceRadius,
        (1.0f - SubsurfaceOpacity) * SubsurfaceMinSourceRadius);
```

サブサーフェスマテリアルでは光が素材内部を散乱するため、
表面よりも広い領域の光を受ける効果をライト半径拡大でシミュレーション。

---

## VirtualShadowMapProjectionComposite のコアロジック

### VirtualShadowMapCompositeTileVS / CompositePS

```hlsl
void VirtualShadowMapCompositeTileVS(in uint InstanceId : SV_InstanceID,
                                      in uint VertexId   : SV_VertexID,
                                      out float4 Position : SV_POSITION)
void VirtualShadowMapCompositePS(in float4 SvPosition : SV_Position,
                                  out float4 OutShadowMask : SV_Target)
```

- `CompositeTileVS`: タイルリストから座標を復元して全画面クワッドを描画
- `CompositePS`: `InputShadowFactor`（float2）を `EncodeLightAttenuation` でエンコードして出力

```hlsl
// bModulateRGB=1 の場合: 直接 float4(ShadowFactor.x, ...) として出力
// それ以外: EncodeLightAttenuation() で GBuffer と整合する形式にエンコード
OutShadowMask = EncodeLightAttenuation(float4(ShadowFactor.x, ShadowFactor.y, ...));
```

### VirtualShadowMapCompositeFromMaskBitsPS

One-Pass Projection で `OutShadowMaskBits` に書き込んだデータを読み取り、  
ライトインデックスに対応するビットを抽出して Shadow Attenuation を生成。

---

## 入出力

### VirtualShadowMapProjection

| リソース | 内容 |
|---------|------|
| `SceneDepthTexture` | SceneDepth（ピクセルの深度）|
| `VirtualShadowMap.PageTable` | 仮想→物理ページマッピングテーブル |
| `PhysicalPagePool` | 物理ページ深度アトラス（シャドウサンプリング元）|
| `HairStrands.HairOnlyDepthTexture` | Hair Strands 用深度（オプション）|
| `OutShadowFactor` | 出力: 影強度（float2: 通常 + サブサーフェス）|
| `OutShadowMaskBits` | 出力: One Pass 用 packed shadow mask（uint4）|

### VirtualShadowMapProjectionComposite

| リソース | 内容 |
|---------|------|
| `InputShadowFactor` | Projection の出力（float2）|
| `ShadowMaskBits` | One Pass Projection の出力（uint4）|
| `OutShadowMask` | Shadow Attenuation テクスチャ（SV_Target0）|

---

## CPU 呼び出しの流れ

```
RenderVirtualShadowMapProjection()        // VirtualShadowMapProjection.cpp:437
  │
  ├─ VirtualShadowMapProjection CS
  │    ├─ One-Pass: 全ローカルライトを1 Dispatch → OutShadowMaskBits
  │    └─ Per-Light: ディレクショナル + 重要ライトを個別に → OutShadowFactor
  │
  └─ CompositeVirtualShadowMapMask()
       └─ VirtualShadowMapCompositeTileVS + CompositePS
          または VirtualShadowMapCompositeFromMaskBitsPS
```
