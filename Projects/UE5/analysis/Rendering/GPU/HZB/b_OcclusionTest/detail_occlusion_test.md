# HZB Occlusion Test シェーダー詳細

- グループ: b - Occlusion Test
- GPU 概要: [[01_hzb_gpu_overview]]
- CPU 詳細: [[b_hzb_occlusion]]
- リファレンス: [[ref_occlusion_test]]

---

## 概要

プリミティブの AABB（軸並行境界ボックス）を FurthestHZB に投影し、  
深度比較によって「可視か不可視か」を GPU 上で判定するパス。  
結果はフレームをまたいで Readback し、CPU の `PrimitiveVisibilityMap` に反映される。

---

## レンダリングパスの構成

```
HZB Occlusion Test パス
  │
  ├─ 入力: BoundsCenterTexture / BoundsExtentTexture（各プリミティブの AABB）
  ├─ 処理: BoxCullFrustum() → AABB を Clip Space に投影
  │          GetScreenRect() → スクリーン矩形を計算
  │          IsVisibleHZB() → FurthestHZB の適切な Mip をサンプル
  └─ 出力: OutColor（1=可視 / 0=不可視）→ 小テクスチャ → CPU Readback
```

---

## 入出力

### 入力

| リソース | 説明 |
|---------|------|
| `BoundsCenterTexture` | AABB 中心座標（各プリミティブ 1 テクセル）|
| `BoundsExtentTexture` | AABB 半径（xyz = extent, w = 0 なら無効）|
| `FurthestHZB`（ View.HZB 経由） | HZB ミップチェーン（読み取り専用）|
| `InvTestTargetSize` | テストターゲットの逆サイズ |

### 出力

| リソース | フォーマット | 内容 |
|---------|------------|------|
| `OutColor` | `float4` | R = 1（可視）/ 0（不可視）|

---

## シェーダーコアロジック（HZBOcclusion.usf）

```hlsl
void HZBTestPS(in float4 SvPosition : SV_Position, out float4 OutColor : SV_Target0)
{
    float2 InUV = SvPosition.xy * InvTestTargetSize;

    float4 BoundsCenter = BoundsCenterTexture.SampleLevel(BoundsCenterSampler, InUV, 0);
    float4 BoundsExtent = BoundsExtentTexture.SampleLevel(BoundsExtentSampler, InUV, 0);

    // w == 0 は無効なプリミティブ（描画なし）
    if (BoundsExtent.w == 0) { OutColor = float4(1, 0, 0, 0); return; }

    // 1. フラスタムカリング（AABB vs View Frustum）
    FFrustumCullData Cull = BoxCullFrustum(
        BoundsCenter.xyz, BoundsExtent.xyz,
        View.TranslatedWorldToClip,
        View.ViewToClip,
        bIsOrtho, true, false);

    // 2. HZB サンプリング（フラスタム内のみ）
    if (Cull.bIsVisible && !Cull.bCrossesNearPlane)
    {
        FScreenRect Rect = GetScreenRect(int4(0, 0, HZBViewSize), Cull, 4);
        Cull.bIsVisible = IsVisibleHZB(Rect, true);  // FurthestHZB をサンプル
    }

    OutColor = Cull.bIsVisible ? 1 : 0;
}
```

### IsVisibleHZB の動作（NaniteHZBCull.ush より）

```hlsl
bool IsVisibleHZB(FScreenRect Rect, bool bUseFurthestHZB)
{
    // Rect のサイズからサンプルする Mip レベルを決定
    float Level = ceil(log2(max(Rect.Extent.x, Rect.Extent.y)));

    // FurthestHZB の4隅をサンプル（最小 DeviceZ = 最遠深度）
    float MinDepth = min4(HZBTexture.SampleLevel(Sampler, UV, Level));

    // プリミティブの最近点（最大 DeviceZ）と比較
    // Reverse-Z: MinDepth < BoundsMaxZ → 遮蔽されていない（可視）
    return BoundsMaxDeviceZ >= MinDepth;
}
```

---

## CPU 呼び出しの流れ

```
FHZBOcclusionTester::Submit()           // HZBOcclusion.cpp
  │
  ├─ BoundsCenter/Extent テクスチャを生成（各プリミティブ 1 ピクセル）
  ├─ AddPass(RDG_EVENT_NAME("HZBOcclusionTest"))
  │   PS: HZBTestPS() → OutColor テクスチャ
  └─ EnqueueCopy(ResultBuffer, OutColor) → CPU Readback バッファ

1フレーム後:
FHZBOcclusionTester::MapResults()
  ├─ ResultBuffer を Map()
  ├─ OutColor.r を読み取り（0 or 1）
  └─ PrimitiveSceneInfo->SetLastRenderTime() → PrimitiveVisibilityMap 更新
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `HZBOcclusion.usf` | PS | AABB の HZB テスト本体 |
| `Nanite/NaniteHZBCull.ush` | ヘッダ | `IsVisibleHZB()` / `BoxCullFrustum()` / `GetScreenRect()` |
| `HZB.usf` | CS/PS | FurthestHZB 生成（テスト対象の HZB）|
