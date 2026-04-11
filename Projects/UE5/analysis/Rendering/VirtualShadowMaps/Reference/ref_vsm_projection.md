# REF: VirtualShadowMapProjection.h / .cpp

- 対象ファイル: `Private/VirtualShadowMaps/VirtualShadowMapProjection.h` / `.cpp`
- 関連Details: [[d_vsm_projection]]

---

## 構造体

### `FTiledVSMProjection`

タイルベース投影最適化用のインダイレクトデータ。

```cpp
struct FTiledVSMProjection
{
    FRDGBufferRef DrawIndirectParametersBuffer;      // ドローコール間接パラメータ
    FRDGBufferRef DispatchIndirectParametersBuffer;  // ディスパッチ間接パラメータ
    FRDGBufferRef TileListDataBufferSRV;             // タイルリストデータ（SRV）
    uint32        TileSize;                          // タイルサイズ（テクセル）
};
// 使用時: タイルに当たるシャドウがあるタイルのみ処理
// → Indirect Draw/Dispatch でシャドウを受けないタイルの処理を完全スキップ
```

---

## 主要 enum

```cpp
enum class EVirtualShadowMapProjectionInputType
{
    GBuffer     = 0,  // 通常ジオメトリ（GBufferから深度・法線を取得）
    HairStrands = 1,  // ヘアストランド（ヘア専用の深度バッファを使用）
};
```

---

## 主要関数

### クリップマップ（ディレクショナルライト）投影

```cpp
void RenderVirtualShadowMapProjection(
    FRDGBuilder& GraphBuilder,
    const FMinimalSceneTextures& SceneTextures,
    const FViewInfo& View,
    int32 ViewIndex,
    FVirtualShadowMapArray& VirtualShadowMapArray,
    const FIntRect ScissorRect,
    EVirtualShadowMapProjectionInputType InputType,
    const TSharedPtr<FVirtualShadowMapClipmap>& Clipmap,
    bool bModulateRGB,
    FTiledVSMProjection* TiledVSMProjection,
    FRDGTextureRef OutputShadowMaskTexture,
    const TSharedPtr<FVirtualShadowMapClipmap>& FirstPersonClipmap = nullptr)
```

- `bModulateRGB`: RGB チャンネルに掛け算で合成するか（デフォルト: 加算合成）
- `FirstPersonClipmap`: 一人称シャドウが別クリップマップの場合に指定
- `TiledVSMProjection`: タイル最適化用データ（nullptr = 全画面処理）

---

### ローカルライト（スポット・ポイント）投影

```cpp
void RenderVirtualShadowMapProjection(
    FRDGBuilder& GraphBuilder,
    const FMinimalSceneTextures& SceneTextures,
    const FViewInfo& View,
    int32 ViewIndex,
    FVirtualShadowMapArray& VirtualShadowMapArray,
    const FIntRect ScissorRect,
    EVirtualShadowMapProjectionInputType InputType,
    const FLightSceneInfo& LightSceneInfo,
    int32 VirtualShadowMapId,
    FRDGTextureRef OutputShadowMaskTexture)
```

---

### マスクビット作成

```cpp
FRDGTextureRef CreateVirtualShadowMapMaskBits(
    FRDGBuilder& GraphBuilder,
    const FMinimalSceneTextures& SceneTextures,
    FVirtualShadowMapArray& VirtualShadowMapArray,
    const TCHAR* Name)
// 複数ライトのシャドウをビット圧縮形式で保持するテクスチャを生成
// → 1テクセルに複数ライト分のシャドウビットを格納
```

---

### ワンパス投影（全ライト一括）

```cpp
void RenderVirtualShadowMapProjectionOnePass(
    FRDGBuilder& GraphBuilder,
    const FMinimalSceneTextures& SceneTextures,
    const FViewInfo& View,
    int32 ViewIndex,
    FVirtualShadowMapArray& VirtualShadowMapArray,
    EVirtualShadowMapProjectionInputType InputType,
    FRDGTextureRef ShadowMaskBits)
// 全VSMを1パスでShadowMaskBitsに書き込む
// → ライト数が多い場合はこちらを使うとタイルキャッシュ効率が上がる
```

---

### マスク合成

```cpp
void CompositeVirtualShadowMapMask(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FIntRect ScissorRect,
    const FRDGTextureRef Input,
    bool bDirectionalLight,
    bool bModulateRGB,
    FTiledVSMProjection* TiledVSMProjection,
    FRDGTextureRef OutputShadowMaskTexture)
// 中間マスクテクスチャを最終ShadowMaskTextureに合成
// bDirectionalLight: ディレクショナルか否か（シェーダーパーミュテーション選択）
```

```cpp
void CompositeVirtualShadowMapFromMaskBits(
    FRDGBuilder& GraphBuilder,
    const FMinimalSceneTextures& SceneTextures,
    const FViewInfo& View,
    int32 ViewIndex,
    const FIntRect ScissorRect,
    FVirtualShadowMapArray& VirtualShadowMapArray,
    EVirtualShadowMapProjectionInputType InputType,
    int32 VirtualShadowMapId,
    FRDGTextureRef ShadowMaskBits,
    FRDGTextureRef OutputShadowMaskTexture)
// ShadowMaskBitsから特定のVSM IDのビットを抽出して最終テクスチャに合成
```

---

## SMRT パラメータ詳細

SMRT（Stochastic Micro Ray Tracing）は VSM 独自のソフトシャドウ手法。  
各フラグメントで複数の「マイクロレイ」を物理ページプール上でトレースする。

| パラメータ | ローカル | ディレクショナル | 説明 |
|-----------|---------|--------------|------|
| `RayCount` | 7 | 7 | フラグメントあたりのレイ数 |
| `SamplesPerRay` | 8 | 8 | レイあたりのサンプル数 |
| `ExtrapolateMaxSlope` | 0.05 | 5.0 | 深度傾き外挿の上限（ペナンブラ推定） |
| `TexelDitherScale` | 2.0 | 2.0 | テクセルジッタースケール（エイリアシング低減） |
| `MaxRayAngleFromLight` | 0.03 | - | 光源からのレイ最大角度（半影サイズに直結） |
| `MaxSlopeBias` | 50.0 | - | 最大傾きバイアス（セルフシャドウ防止） |
| `RayLengthScale` | - | 1.5 | ディレクショナルレイ長スケール |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Shadow.Virtual.SMRT.RayCountLocal` | 7 | ローカルライト SMRTレイ数 |
| `r.Shadow.Virtual.SMRT.RayCountDirectional` | 7 | ディレクショナル SMRTレイ数 |
| `r.Shadow.Virtual.SMRT.SamplesPerRayLocal` | 8 | ローカル レイあたりサンプル数 |
| `r.Shadow.Virtual.SMRT.SamplesPerRayDirectional` | 8 | ディレクショナル レイあたりサンプル数 |
| `r.Shadow.Virtual.SMRT.MaxRayAngleFromLight` | 0.03 | 最大レイ角度（ソフト半影サイズ） |
| `r.Shadow.Virtual.SMRT.TexelDitherScaleLocal` | 2.0 | ローカルテクセルディザースケール |
| `r.Shadow.Virtual.SMRT.TexelDitherScaleDirectional` | 2.0 | ディレクショナルテクセルディザースケール |
| `r.Shadow.Virtual.SMRT.ExtrapolateMaxSlopeLocal` | 0.05 | ローカル傾き外挿上限 |
| `r.Shadow.Virtual.SMRT.ExtrapolateMaxSlopeDirectional` | 5.0 | ディレクショナル傾き外挿上限 |
| `r.Shadow.Virtual.SMRT.MaxSlopeBiasLocal` | 50.0 | ローカル最大傾きバイアス |
| `r.Shadow.Virtual.SMRT.RayLengthScaleDirectional` | 1.5 | ディレクショナルレイ長スケール |
| `r.Shadow.Virtual.ScreenRayLength` | 0.015 | スクリーンスペースレイ長 |
| `r.Shadow.Virtual.TranslucentQuality` | 0 | 半透明投影品質（0=低, 1=高） |
| `r.Shadow.Virtual.SubsurfaceShadowMinSourceAngle` | 5 | サブサーフェース最小光源角度（度） |
| `r.Shadow.Virtual.CullBackfacingPixels` | 1 | 裏面向きピクセルのシャドウカリング |
| `r.Shadow.Virtual.ForcePerLightShadowMaskClear` | 0 | デバッグ用マスクバッファクリア |
| `r.Shadow.Virtual.Visualize.ShowCachedPagesOnly` | 0 | キャッシュページのみ可視化 |

---

> [!note]- SMRT の ExtrapolateMaxSlope によるペナンブラ推定
> SMRT では深度サンプルの傾き（`ExtrapolateMaxSlope`）を使ってオクルーダーの「厚み」を推定し、半影領域を計算する。  
> ディレクショナルライトは `ExtrapolateMaxSlopeDirectional = 5.0`（大きい）、ローカルライトは `0.05`（小さい）。  
> ディレクショナルライトは遠距離を扱うため深度差が大きく許容値を大きくしている。  
> この値が小さすぎると半影が出なくなり、大きすぎると不自然に広い影になる。

> [!note]- OnePass vs PerLight 投影の使い分け
> `RenderVirtualShadowMapProjectionOnePass()` は多数のローカルライトをタイルベースで一括処理する最適化パス。  
> タイルごとにどのライトが影響するかを Tiled Deferred Lighting と同様に判別し、  
> 影響するライトのシャドウのみ `ShadowMaskBits` の対応ビットに書き込む。  
> ライト数が少ない場合や透明オブジェクトには `PerLight` 版が使われる。

> [!note]- EVirtualShadowMapProjectionInputType と HairStrands
> `EVirtualShadowMapProjectionInputType::HairStrands` を指定すると、GBuffer の代わりにヘアストランド専用の  
> 深度バッファと法線を使ってシャドウをサンプリングする。  
> ヘアは独自の深度解決が必要なため（複数の半透明レイヤー）、専用パスが存在する。  
> 通常の不透明ジオメトリには `GBuffer` を使う。
