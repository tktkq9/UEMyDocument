# a: HZB 構築（BuildHZB）

- 対象ファイル: `HZB.h/.cpp`
- 概要: [[21_hzb_overview]]

---

## 概要

`BuildHZB()` は **SceneDepth テクスチャからミップチェーンを持つ HZB テクスチャを生成**する。  
FurthestHZB（各ブロックの最大深度）と ClosestHZB（最小深度）を並行して構築し、  
FViewInfo に格納することで後続パス（SSR / Lumen / Occlusion / Nanite）が利用できる。

---

## HZB テクスチャ仕様

```
解像度:
  Mip0 = NextPowerOfTwo(ViewWidth/2) × NextPowerOfTwo(ViewHeight/2)
  Mip1 = Mip0 / 2
  ...
  MipN = 1×1 （最終ミップ）

フォーマット:
  FurthestHZB: PF_R16F（R チャンネル = 最大深度）
  ClosestHZB:  PF_R16F（R チャンネル = 最小深度）

  ※ ClosestHZB は一部プラットフォームのみ生成（IsHZBValid チェックが必要）

サンプラー:
  FurthestHZB: Point Clamp（正確な最大深度参照）
  ClosestHZB:  Point Clamp
```

---

## BuildHZB() フロー（HZB.cpp）

```
BuildHZB(FRDGBuilder& GraphBuilder, FViewInfo& View)
  │
  ├─ [A] 解像度計算
  │   HZBSize = (NextPOT(ViewWidth/2), NextPOT(ViewHeight/2))
  │   MipCount = FloorLog2(Max(HZBSize.X, HZBSize.Y)) + 1
  │
  ├─ [B] HZBTexture 確保（RDG）
  │   FurthestHZBTexture: PF_R16F, MipCount, UAV 対応
  │   ClosestHZBTexture:  PF_R16F, MipCount, UAV 対応（サポートプラットフォームのみ）
  │
  ├─ [C] Mip0 生成（SceneDepth → HZB Mip0）
  │   HZBBuildVS / HZBBuildPS（または CS）
  │   → SceneDepth を 1/2 にダウンサンプル
  │   → FurthestHZB: Max(4近傍深度)
  │   → ClosestHZB:  Min(4近傍深度)
  │
  ├─ [D] Mip1〜N 生成（Reduce ループ）
  │   for each Mip in [1, MipCount):
  │     HZBBuildCS（Compute Shader）
  │     → 前ミップの 2×2 テクセルから Max / Min を計算
  │     → 結果を現在ミップに書き込み
  │     DispatchSize = (Ceil(MipWidth/8), Ceil(MipHeight/8), 1)
  │
  └─ [E] FViewInfo に格納
      View.HZB = FurthestHZBTexture
      View.ClosestHZB = ClosestHZBTexture
      → GetHZBParameters() でシェーダーパラメータを生成

【深度値の意味】
  UE5 の深度は Reverse-Z（遠方が 0, 近傍が 1）
  FurthestHZB: 遠方の深度値が小さい → ミップ内の「最小値」= 最も遠い深度
  ClosestHZB:  近傍の深度値が大きい → ミップ内の「最大値」= 最も近い深度

  ※ Reverse-Z では「FurthestHZB の Min Reduce」が最も遠いオブジェクトを保持
```

---

## HZB の利用パターン

```
[SSR（Screen Space Reflection）]
  ReflectionEnvironment.usf:
    HZBTexture.SampleLevel(UV, MipLevel) で階層的にレイトレース
    → 反射レイが深度バッファに衝突する位置を Mip 降順で求める

[Lumen Screen Probe]
  LumenScreenProbeHierarchical.usf:
    HZB の各ミップレベルをサンプルして空きスペースを検出
    → Probe の配置位置を決定

[Nanite Persistent Cull]
  NaniteCulling.usf:
    前フレームの HZB を参照して Cluster の可視判定
    → HZB に Cluster の AABB が全て隠れていれば Culled

[Shadow Map Cull]
  CullShadowDepthMaps.usf:
    HZB で遮蔽されているライスト範囲をスキップ
```

---

## 関連リファレンス

- [[ref_hzb_resources]] — `FHZBParameters` / `EHZBType` / テクスチャ仕様
- [[b_hzb_occlusion]] — HZB Occlusion Culling フロー
