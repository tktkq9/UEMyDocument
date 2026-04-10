# VSM d: VirtualShadowMapProjection + Shaders（投影・シェーダー基盤）

- 対象: `VirtualShadowMaps/VirtualShadowMapProjection.h/.cpp` / `VirtualShadowMapShaders.h`
- 上位: [[04_vsm_overview]]

---

## 役割

**投影パス（Projection）**: ライティングパスで VSM をサンプリングし、シャドウマスクを生成する。  
**シェーダー基盤（Shaders）**: VSM 用グローバルシェーダーの基底クラスと定数定義。

---

## 投影パスの概要

```
ライティングパス
  ↓
RenderVirtualShadowMapProjection()
  → 物理ページプールをサンプリング
  → SMRT（Stochastic Micro Ray Tracing）で PCF 品質のソフトシャドウを生成
  → ShadowMaskTexture に書き込み
  ↓
CompositeVirtualShadowMapMask()
  → 最終的な ShadowMask に合成
```

---

## SMRT（Stochastic Micro Ray Tracing）

VSM 独自のソフトシャドウ手法。  
物理ページプール上で「マイクロレイ」をストキャスティックに複数本トレースし、  
ポイントサンプリングを超えた半影表現を実現する。

```
各フラグメントで:
  1. レイ数（RayCount）本のレイを生成
  2. 各レイで SamplesPerRay 個のサンプルを採取
  3. ソフトシャドウ率を集計 → 0.0〜1.0 のシャドウ値

ローカルライト用（スポット・ポイント）:
  → r.Shadow.Virtual.SMRT.RayCountLocal（デフォルト7）
  → r.Shadow.Virtual.SMRT.SamplesPerRayLocal（デフォルト8）

ディレクショナルライト用:
  → r.Shadow.Virtual.SMRT.RayCountDirectional（デフォルト7）
  → r.Shadow.Virtual.SMRT.SamplesPerRayDirectional（デフォルト8）
```

---

## 主要関数

### クリップマップ（ディレクショナルライト）投影

```cpp
void RenderVirtualShadowMapProjection(
    FRDGBuilder&,
    const FMinimalSceneTextures&,
    const FViewInfo&, int32 ViewIndex,
    FVirtualShadowMapArray&,
    const FIntRect ScissorRect,
    EVirtualShadowMapProjectionInputType InputType,
    const TSharedPtr<FVirtualShadowMapClipmap>&,
    bool bModulateRGB,
    FTiledVSMProjection* TiledVSMProjection,
    FRDGTextureRef OutputShadowMaskTexture,
    const TSharedPtr<FVirtualShadowMapClipmap>& FirstPersonClipmap)
```

### ローカルライト（スポット・ポイント）投影

```cpp
void RenderVirtualShadowMapProjection(
    FRDGBuilder&,
    const FMinimalSceneTextures&,
    const FViewInfo&, int32 ViewIndex,
    FVirtualShadowMapArray&,
    const FIntRect ScissorRect,
    EVirtualShadowMapProjectionInputType InputType,
    const FLightSceneInfo&,
    int32 VirtualShadowMapId,
    FRDGTextureRef OutputShadowMaskTexture)
```

### ワンパス投影（全ライト一括）

```cpp
void RenderVirtualShadowMapProjectionOnePass(
    FRDGBuilder&,
    const FMinimalSceneTextures&,
    const FViewInfo&, int32 ViewIndex,
    FVirtualShadowMapArray&,
    EVirtualShadowMapProjectionInputType InputType,
    FRDGTextureRef ShadowMaskBits)
// 全ライトをビットマスクバッファに一括処理（タイルベース最適化用）
```

### マスクビット管理

```cpp
FRDGTextureRef CreateVirtualShadowMapMaskBits(
    FRDGBuilder&,
    const FMinimalSceneTextures&,
    FVirtualShadowMapArray&,
    const TCHAR* Name)
// ライトごとのシャドウを圧縮ビット形式で保持するテクスチャを作成

void CompositeVirtualShadowMapMask(
    FRDGBuilder&,
    const FViewInfo&,
    const FIntRect ScissorRect,
    const FRDGTextureRef Input,
    bool bDirectionalLight,
    bool bModulateRGB,
    FTiledVSMProjection*,
    FRDGTextureRef OutputShadowMaskTexture)
// ビットマスクを最終シャドウマスクに合成

void CompositeVirtualShadowMapFromMaskBits(...)
// ShadowMaskBits から特定VSMのマスクを抽出して合成
```

---

## 入力タイプ

```cpp
enum class EVirtualShadowMapProjectionInputType
{
    GBuffer      = 0,   // 通常ジオメトリ（GBufferから深度・法線取得）
    HairStrands  = 1,   // ヘアストランド（独自深度バッファ使用）
};
```

---

## タイル投影最適化

```cpp
struct FTiledVSMProjection
{
    FRDGBufferRef DrawIndirectParametersBuffer;
    FRDGBufferRef DispatchIndirectParametersBuffer;
    FRDGBufferRef TileListDataBufferSRV;
    uint32 TileSize;
};
// タイル単位で投影処理を Indirect Dispatch することで
// シャドウを受けないタイルの処理を完全スキップ
```

---

## シェーダー基底クラス（VirtualShadowMapShaders.h）

```cpp
// VSM 用グローバルシェーダーの基底
class FVirtualShadowMapGlobalShader : public FGlobalShader
{
    // ShouldCompilePermutation(): Renderer モジュールの場合のみコンパイル
    // ModifyCompilationEnvironment(): HLSL 2021 を有効化
};

// VSM ページ管理コンピュートシェーダーの基底
class FVirtualShadowMapPageManagementShader : public FVirtualShadowMapGlobalShader
{
    static constexpr int32 DefaultCSGroupXY = 8;   // 2D ディスパッチ用グループサイズ
    static constexpr int32 DefaultCSGroupX  = 256; // 1D ディスパッチ用グループサイズ
    // ModifyCompilationEnvironment(): グループサイズ定数をシェーダーに注入
};
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Shadow.Virtual.SMRT.RayCountLocal` | 7 | ローカルライト SMRTレイ数 |
| `r.Shadow.Virtual.SMRT.RayCountDirectional` | 7 | ディレクショナル SMRTレイ数 |
| `r.Shadow.Virtual.SMRT.SamplesPerRayLocal` | 8 | ローカルライト レイあたりサンプル数 |
| `r.Shadow.Virtual.SMRT.SamplesPerRayDirectional` | 8 | ディレクショナル レイあたりサンプル数 |
| `r.Shadow.Virtual.SMRT.MaxRayAngleFromLight` | 0.03 | 光源からのレイ最大角度（ソフト半影サイズ） |
| `r.Shadow.Virtual.SMRT.TexelDitherScaleLocal` | 2.0 | ローカルテクセルディザースケール |
| `r.Shadow.Virtual.SMRT.TexelDitherScaleDirectional` | 2.0 | ディレクショナルテクセルディザースケール |
| `r.Shadow.Virtual.SMRT.ExtrapolateMaxSlopeLocal` | 0.05 | ローカル傾き外挿上限 |
| `r.Shadow.Virtual.SMRT.ExtrapolateMaxSlopeDirectional` | 5.0 | ディレクショナル傾き外挿上限 |
| `r.Shadow.Virtual.SMRT.MaxSlopeBiasLocal` | 50.0 | ローカル最大傾きバイアス |
| `r.Shadow.Virtual.SMRT.RayLengthScaleDirectional` | 1.5 | ディレクショナルレイ長スケール |
| `r.Shadow.Virtual.ScreenRayLength` | 0.015 | スクリーンスペースレイ長 |
| `r.Shadow.Virtual.TranslucentQuality` | 0 | 半透明投影品質（0/1） |
| `r.Shadow.Virtual.SubsurfaceShadowMinSourceAngle` | 5 | サブサーフェース最小光源角度 |
| `r.Shadow.Virtual.CullBackfacingPixels` | 1 | 裏面ピクセルのシャドウカリング |
| `r.Shadow.Virtual.Visualize.Layout` | 0 | 可視化レイアウト（0=全画面,1=サムネイル,2=分割） |
| `r.Shadow.Virtual.Stats.Visible` | 0 | デバッグ統計表示 |
