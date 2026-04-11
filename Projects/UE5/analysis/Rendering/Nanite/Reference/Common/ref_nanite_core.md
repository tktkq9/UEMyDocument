# リファレンス：Nanite.h / Nanite.cpp

- グループ: Common
- 上位: [[03_nanite_overview]]
- 関連: [[ref_nanite_shared]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/Nanite.h/.cpp`

---

## 概要

Nanite レンダリングパイプライン全体のエントリポイント。  
`FSharedContext`・`FRasterContext`・`FRasterResults` を定義し、  
シャドウマップ・シャドウキューブマップ等の発行関数を宣言する。

---

## FSharedContext

全レンダリングパスで共有するフレーム共通設定。

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `RenderFlags` | `uint32` | レンダリングフラグ一式 |
| `RasterScheduling` | `ERasterScheduling` | HW Only / HW+SW / Overlap |
| `ShaderMap` | `FGlobalShaderMap*` | グローバルシェーダーマップ |
| `Pipeline` | `EPipeline` | Primary / Shadows / Lumen / Editor / HitProxy |

---

## FRasterContext

パスごとに `InitRasterContext()` で生成するラスタライズコンテキスト。

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `VisBuffer64` | `FRDGTextureRef` | Depth(40bit) + MaterialDepth(24bit) |
| `DbgBuffer64` | `FRDGTextureRef` | デバッグバッファ |
| `ShadingMaskBuffer` | `FRDGBufferRef` | シェーディングマスク（ビン可視判定）|
| `RasterScheduling` | `ERasterScheduling` | スケジューリング戦略 |
| `OutputBufferMode` | `EOutputBufferMode` | VisBuffer / DepthOnly |
| `RasterBinCount` | `uint32` | 登録済みラスタライズビン数 |
| `bCustomPass` | `bool` | カスタムパスか |

---

## FRasterResults

カリング・ラスタライズ完了後の結果バッファ群。

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `PageRequests` | `FRDGBufferRef` | ページストリーミング要求 |
| `VisiblePatches` | `FRDGBufferRef` | テッセレーション用可視パッチリスト |
| `VisibilityResults` | `FNaniteVisibilityResults` | ラスタ/シェーディングビン可視性ビットマスク |
| `ViewsCount` | `uint32` | 処理したビュー数 |

---

## FConfiguration

| フラグ | 説明 |
|-------|------|
| `bUpdateStreaming : 1` | ストリーミング更新を行うか |
| `bSupportsMultiplePasses : 1` | 複数パス（シャドウ等）対応か |
| `bAllowProgrammableRaster : 1` | プログラマブルラスタ許可か |

---

## IRenderer — 仮想インターフェース

```cpp
class IRenderer
{
public:
    static TUniquePtr<IRenderer> Create(
        FRDGBuilder&, const FScene&, const FViewInfo&,
        FSceneUniformBuffer&, const FSharedContext&, const FRasterContext&,
        const FConfiguration&, const FIntRect& ViewRect,
        const FRDGTextureRef PrevHZB,
        FVirtualShadowMapArray* = nullptr);

    virtual void DrawGeometry(...) = 0;   // 2パスカリング + HW/SW ラスタ
    virtual void ExtractResults(FRasterResults&) = 0; // 結果の書き出し
};
```

---

## 主要関数

| 関数 | 説明 |
|------|------|
| `Nanite::EmitShadowMap(...)` | Nanite プリミティブのシャドウデプスを生成 |
| `Nanite::EmitCubemapShadow(...)` | キューブマップシャドウを生成 |
| `Nanite::PrintStats(...)` | GPU 統計情報をスクリーンに出力 |
| `Nanite::ExtractShadingDebug(...)` | シェーディングデバッグ情報を抽出 |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Nanite.ShowStats` | 0 | GPU 統計情報の画面表示 |
| `r.Nanite.StatsFilter` | -1 | 統計フィルタ（-1=全て） |
| `r.Nanite.Shadows` | 1 | Nanite シャドウ有効/無効 |

---

> [!note]- IRenderer::Create の設計意図
> `IRenderer::Create()` は `FSharedContext`（パス共通設定）と `FRasterContext`（フレーム一時バッファ）を受け取り、具象クラス `FRenderer` を `TUniquePtr<IRenderer>` として返す。  
> この設計により `DrawGeometry()` の呼び出し元は具体的なカリング・ラスタ実装を知らなくてよい。  
> `FSharedContext::Pipeline` を変えるだけで Primary / Shadows / Lumen 等の異なるパスに対応できる。

> [!note]- FRasterContext のライフサイクル
> `InitRasterContext()` はフレームごとに呼ばれ、RDG テクスチャを新規作成する（フレーム一時）。  
> `VisBuffer64` は `R64_UINT` フォーマット（上位 40bit = Depth、下位 24bit = MaterialDepth）。  
> `EOutputBufferMode::DepthOnly` のシャドウパスでは `VisBuffer64` は確保せず深度テクスチャのみを使う。

> [!note]- EPipeline と ERasterScheduling の組み合わせ
> `EPipeline::Shadows` と `EPipeline::Lumen` では通常 `ERasterScheduling::HardwareAndSoftwareOverlap` が無効化される（非同期コンピュートのオーバーヘッドが利点を上回るため）。  
> `r.Nanite.AsyncRasterization.ShadowDepths` / `r.Nanite.AsyncRasterization.LumenMeshCards` で個別に制御できる。
