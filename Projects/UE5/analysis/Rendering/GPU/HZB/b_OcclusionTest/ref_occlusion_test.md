# REF: HZB Occlusion Test シェーダー

- グループ: b - Occlusion Test
- 詳細: [[detail_occlusion_test]]
- CPU リファレンス: [[ref_hzb_occlusion]]
- ソース: `Engine/Shaders/Private/HZBOcclusion.usf`  
          `Engine/Shaders/Private/Nanite/NaniteHZBCull.ush`

---

## HZBTestPS（HZBOcclusion.usf）

### エントリポイント

```hlsl
void HZBTestPS(
    in float4 SvPosition : SV_Position,
    out float4 OutColor  : SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Pixel Shader |
| **目的** | プリミティブの AABB を HZB に投影して可視判定 |
| **出力** | `OutColor.r = 1`（可視）/ `0`（不可視）|

#### 入力バインド

| バインド名 | 型 | 説明 |
|-----------|------|------|
| `BoundsCenterTexture` | `Texture2D` | AABB 中心（+PreViewTranslation 済み）|
| `BoundsExtentTexture` | `Texture2D` | AABB 半径（w=0 で無効マーク）|
| `InvTestTargetSize` | `float2` | テストターゲット逆サイズ |
| `HZBTexture`（View 経由）| `Texture2D` | FurthestHZB ミップチェーン |

#### CPU バインド

```cpp
// HZBOcclusion.cpp - FHZBOcclusionTester::Submit()
FHZBOcclusionTestParameters* PassParameters = ...;
PassParameters->BoundsCenterTexture = BoundsCenterSRV;
PassParameters->BoundsExtentTexture = BoundsExtentSRV;
PassParameters->InvTestTargetSize = FVector2f(1.0f / Width, 1.0f / Height);

GraphBuilder.AddPass(
    RDG_EVENT_NAME("HZBOcclusionTest"),
    PassParameters,
    ERDGPassFlags::Raster,
    [](FRHICommandList& RHICmdList)
    {
        // DrawIndexedPrimitive でフルスクリーンクワッド
        // RT: ResultTexture（R8）
    });
```

---

## IsVisibleHZB（NaniteHZBCull.ush）

```hlsl
bool IsVisibleHZB(FScreenRect Rect, bool bUseFurthestHZB)
// HZB の適切な Mip をサンプルして AABB の可視性を判定
// Rect.Extent でサンプルする Mip を決定（ceil(log2(MaxExtent))）
```

---

## BoxCullFrustum（NaniteHZBCull.ush）

```hlsl
FFrustumCullData BoxCullFrustum(
    float3 BoundsCenter,
    float3 BoundsExtent,
    float4x4 TranslatedWorldToClip,
    float4x4 ViewToClip,
    bool bIsOrtho,
    bool bNearClip,
    bool bSkipCulling)
// AABB 8 頂点を Clip Space に変換してフラスタムと交差判定
// FFrustumCullData.bIsVisible / bCrossesNearPlane を返す
```

---

## FScreenRect（NaniteHZBCull.ush）

```hlsl
struct FScreenRect
{
    int4   Pixels;    // ピクセル境界（inclusive）
    float4 UVs;       // UV 境界
    float2 Extent;    // XY サイズ
    float  MinZ;      // 最小デバイスZ（最近点）
};

FScreenRect GetScreenRect(
    int4 ViewRect,
    FFrustumCullData Cull,
    int SubpixelBits)
// Clip Space → Screen Space へ変換してスクリーン矩形を生成
```

---

## PrimitiveVisibilityMap 更新フロー

```cpp
// FHZBOcclusionTester::MapResults() (HZBOcclusion.cpp)
void MapResults(FRHICommandListImmediate& RHICmdList, const FViewInfo& View)
{
    // Readback バッファを Map
    const uint8* ResultData = (uint8*)RHICmdList.LockBuffer(ResultBuffer, ...);

    for (int32 Index = 0; Index < Primitives.Num(); Index++)
    {
        bool bVisible = ResultData[Index] > 0;
        FPrimitiveSceneInfo* PrimitiveSceneInfo = Primitives[Index];

        if (bVisible)
            PrimitiveSceneInfo->LastRenderTime = CurrentTime;
        // bVisible = false → 次フレームは描画をスキップ
    }
    RHICmdList.UnlockBuffer(ResultBuffer);
}
```

---

> [!note]- 1フレーム遅延の扱い
> HZB Occlusion Test の結果は **1フレーム後** に CPU へ戻る。そのため、カメラが急移動した場合に一瞬ポッピングが起きる可能性がある。  
> UE5 では `r.HZBOcclusion.SkipLastFrameOutdatedTests=1` でビュー行列が大きく変わったフレームの古い結果を無視することでこれを軽減する。
