# ref: MegaLights デノイジング クラス群

- 対象: `MegaLightsViewState.h` / `MegaLightsDenoising.cpp`
- Details: [[d_megalights_resolve]]

---

## FMegaLightsViewState（フレーム間ヒストリ）

```cpp
// MegaLightsViewState.h:11
class FMegaLightsViewState
{
public:
    struct FResources
    {
        // ---- ライティング HistoryTexture ----
        TRefCountPtr<IPooledRenderTarget> DiffuseLightingHistory;    // 前フレームの拡散光
        TRefCountPtr<IPooledRenderTarget> SpecularLightingHistory;   // 前フレームの鏡面光
        TRefCountPtr<IPooledRenderTarget> LightingMomentsHistory;    // 2次モーメント（分散計算用）
        TRefCountPtr<IPooledRenderTarget> NumFramesAccumulatedHistory; // 蓄積フレーム数

        // ---- ライストハッシュバッファ ----
        // 前フレームに可視だったライストのハッシュ（ライスト変化検出用）
        TRefCountPtr<FRDGPooledBuffer> VisibleLightHashHistory;
        TRefCountPtr<FRDGPooledBuffer> VisibleLightMaskHashHistory;
        TRefCountPtr<FRDGPooledBuffer> VolumeVisibleLightHashHistory;
        TRefCountPtr<FRDGPooledBuffer> TranslucencyVolume0VisibleLightHashHistory;
        TRefCountPtr<FRDGPooledBuffer> TranslucencyVolume1VisibleLightHashHistory;

        // ---- シーン深度・法線ヒストリ（オプション） ----
        // StochasticLightingViewState の共有 History がない場合に使用
        TRefCountPtr<IPooledRenderTarget> SceneDepthHistory;
        TRefCountPtr<IPooledRenderTarget> SceneNormalHistory;

        // ---- タイルグリッド情報 ----
        FIntPoint HistoryVisibleLightHashViewMinInTiles  = 0;
        FIntPoint HistoryVisibleLightHashViewSizeInTiles = 0;
        FIntVector HistoryVolumeVisibleLightHashViewSizeInTiles    = FIntVector::ZeroValue;
        FIntVector HistoryTranslucencyVolumeVisibleLightHashSizeInTiles = FIntVector::ZeroValue;

        void SafeRelease();
        uint64 GetGPUSizeBytes(bool bLogSizes) const;
    };

    // GBuffer パスのヒストリ
    FResources GBuffer;
    // Hair Strands パスのヒストリ（GBuffer とは独立して管理）
    FResources HairStrands;

    void SafeRelease();
    uint64 GetGPUSizeBytes(bool bLogSizes) const;
};
```

---

## ヒストリの読み書きフロー

```
[読み込み（MegaLights.cpp:1380～）]
  View.ViewState->MegaLights.GBuffer
    .DiffuseLightingHistory   → DiffuseLightingHistory テクスチャに登録
    .SpecularLightingHistory  → SpecularLightingHistory テクスチャに登録
    .LightingMomentsHistory   → LightingMomentsHistory テクスチャに登録
    .NumFramesAccumulatedHistory → NumFramesAccumulatedHistory テクスチャに登録
    .VisibleLightHashHistory  → VisibleLightHashHistory バッファに登録

[書き込み（MegaLightsDenoising.cpp:422～）]
  GraphBuilder.QueueTextureExtraction(DiffuseLighting,      &ViewState.DiffuseLightingHistory)
  GraphBuilder.QueueTextureExtraction(SpecularLighting,     &ViewState.SpecularLightingHistory)
  GraphBuilder.QueueTextureExtraction(LightingMoments,      &ViewState.LightingMomentsHistory)
  GraphBuilder.QueueTextureExtraction(NumFramesAccumulated, &ViewState.NumFramesAccumulatedHistory)
  GraphBuilder.QueueBufferExtraction(VisibleLightHash,      &ViewState.VisibleLightHashHistory)
  → フレーム終了後に IPooledRenderTarget へ抽出
```

---

## テンポラルデノイズのアルゴリズム

```
1. History テクスチャをリプロジェクション（前フレームのスクリーン位置を計算）
2. VisibleLightHash で前フレームと現フレームのライスト構成を比較
   → ハッシュ不一致 → History Miss（BlendFactor を大きく取り速リセット）
3. Neighborhood Clamp（近傍ピクセルの輝度でゴースト抑制）
   → r.MegaLights.Temporal.NeighborhoodClampScale で強度調整
4. NumFramesAccumulated 更新
   - History Miss → MinFramesAccumulated（デフォルト 4）にリセット
   - 通常 → min(NumFrames + 1, MaxFramesAccumulated)（デフォルト 12）
5. BlendFactor = 1.0 / NumFramesAccumulated でブレンド
   → 小さいほど過去のフレームを重視（ノイズ少・ゴースト多）
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.MegaLights.Temporal` | 1 | テンポラル蓄積の有効/無効 |
| `r.MegaLights.Temporal.MaxFramesAccumulated` | 12 | 最大蓄積フレーム数 |
| `r.MegaLights.Temporal.MinFramesAccumulated` | 4 | History Miss 時の初期蓄積フレーム数 |
| `r.MegaLights.Temporal.NeighborhoodClampScale` | 1.0 | 近傍クランプ強度 |
| `r.MegaLights.Spatial` | 1 | 空間フィルタの有効/無効 |
| `r.MegaLights.Spatial.DepthWeightScale` | 1.0 | バイラテラルフィルタの深度重み |

---

> [!note]- GBuffer / HairStrands の独立ヒストリ
> `FMegaLightsViewState` は `GBuffer` と `HairStrands` の2つの `FResources` を持つ。
> Hair Strands は解像度・サンプル数が GBuffer と異なるため独立して管理される。
> `MegaLightsDenoising.cpp:412` で `InputType == EMegaLightsInput::HairStrands` を
> 判定して参照先を切り替えている。

> [!note]- VisibleLightHash の役割
> 毎フレームの可視ライストをハッシュ化してバッファに書き込む。
> テンポラル蓄積時に前フレームのハッシュと比較し、ライストが追加/削除/移動した場合に
> NumFramesAccumulated をリセットしてゴーストを防ぐ。

> [!note]- LightingMoments テクスチャ
> 各ピクセルの輝度の1次モーメント（平均）と2次モーメントを格納する。
> これを使って分散（Variance）を計算し、Neighborhood Clamp のクランプ範囲を
> 適応的に決定することでゴーストと残像のトレードオフを最適化する。
