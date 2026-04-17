# b: Shadow Depth Map 描画

- 対象ファイル: `ShadowDepthRendering.h/.cpp`
- 概要: [[17_shadow_overview]]

---

## 概要

`RenderShadowDepthMaps()` がすべての `FProjectedShadowInfo` に対して  
Shadow Depth テクスチャ（深度バッファ）を描画する。  
Nanite メッシュは Compute Shader 経由、非 Nanite は通常の DrawCall を使用する。

---

## FShadowDepthMeshProcessor

```cpp
// ShadowDepthRendering.h（概略）
class FShadowDepthMeshProcessor : public FMeshPassProcessor
{
public:
    FShadowDepthMeshProcessor(
        const FScene* Scene,
        const FSceneView* InViewIfDynamicMeshCommand,
        const FProjectedShadowInfo* InProjectedShadowInfo, // シャドウ固有パラメータ
        bool bInReflectiveShadowMap,
        EMeshPass::Type InMeshPassType,
        FMeshPassDrawListContext* InDrawListContext);

    virtual void AddMeshBatch(...) override;
};
```

---

## Shadow Depth シェーダー

```cpp
// ShadowDepthRendering.h（概略）

// 頂点シェーダー
class FShadowDepthVS : public FMeshMaterialShader
{
    // Shadow View Matrix（TranslatedWorldToClip）で頂点変換
    // CSM: 光源方向に向いたオルソグラフィック投影
    // Spot/Point: 光源位置からのパースペクティブ投影
};

// ピクセルシェーダー（Masked / PDO あり時のみ）
class FShadowDepthPS : public FMeshMaterialShader
{
    // Masked マテリアル: OpacityMask < ClipValue → discard
    // Pixel Depth Offset: PDO 値で深度を調整して書き込み
};
```

---

## Shadow Atlas テクスチャフォーマット

| ライスト種別 | フォーマット | 説明 |
|----------|----------|------|
| Directional（CSM） | D24S8 or R32_FLOAT | アトラスに複数カスケード格納 |
| Spot Light | D24S8 | 個別テクスチャ or アトラス |
| Point Light（キューブ）| D24S8 | 6面 = 6枚または Cube テクスチャ |
| Per-Object Dynamic | R32_FLOAT | 動的シャドウ（小解像度）|
| Cached Static | D24S8 | キャッシュ済み静的シャドウ |

---

## RenderShadowDepthMaps() フロー

```
RenderShadowDepthMaps()                         ShadowRendering.cpp
  │
  ├─ for each FProjectedShadowInfo:
  │
  │   ── Shadow Atlas バインド ─────────────────
  │   RenderTargets をバインド（アトラス内のビューポートを設定）
  │   ScissorRect = (X, Y, X+ResolutionX, Y+ResolutionY)
  │
  │   ── [Nanite メッシュ] ──────────────────────
  │   Nanite::RenderShadowDepths()
  │     → NaniteShadowDepth CS（Compute Shader）
  │     → VisBuffer（ClusterID + TriangleID）から
  │       Shadow View 座標に変換して深度テクスチャに書き込み
  │     → ラスタライザを通さないため高速
  │
  │   ── [非 Nanite Opaque] ────────────────────
  │   FProjectedShadowInfo::RenderDepth()
  │     FShadowDepthMeshProcessor::AddMeshBatch()
  │       FShadowDepthVS で TranslatedWorldToClip 変換
  │       PS なし（深度のみ書き込み）
  │       → RHIDrawIndexedPrimitive()
  │
  │   ── [非 Nanite Masked] ────────────────────
  │   同上 + FShadowDepthPS でアルファテスト
  │
  └─ [Cached Shadow（静的シャドウ）]
      CacheMode == SDCM_StaticPrimitivesOnly の場合
      → 前フレームの Shadow テクスチャをコピーして再利用
      → 動的オブジェクトのみ追加描画

【アトラス内の座標補正】
  実際のテクセル = (X + BorderSize, Y + BorderSize)
  シェーダー内の Shadow UV:
    UV = (ShadowPos.xy * 0.5 + 0.5) * (ResolutionX / AtlasWidth)
       + vec2(X + BorderSize, Y + BorderSize) / AtlasSize
```

---

## 関連リファレンス

- [[ref_shadow_rendering]] — `FShadowDepthVS` / `FShadowDepthPS` / `FShadowDepthMeshProcessor`
- [[a_shadow_setup]] — `FProjectedShadowInfo` 生成・アトラスパッキング
- [[c_shadow_projection]] — Shadow Projection（深度比較・PCF）
