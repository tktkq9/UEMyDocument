# VSM ドキュメント作成チェックリスト

## Details（グループ別概要）

- [x] `a_vsm_array.md`          … VirtualShadowMapArray（コア管理・ページ割り当て）
- [x] `b_vsm_cache_manager.md`  … VirtualShadowMapCacheManager（キャッシュ・無効化）
- [x] `c_vsm_clipmap.md`        … VirtualShadowMapClipmap（ディレクショナルライト）
- [x] `d_vsm_projection.md`     … VirtualShadowMapProjection + Shaders（投影）

## Reference（ソースファイル対応 5本）

### a_Array グループ（1本）
- [x] `ref_vsm_array.md`          … VirtualShadowMapArray.h/.cpp

### b_CacheManager グループ（1本）
- [x] `ref_vsm_cache_manager.md`  … VirtualShadowMapCacheManager.h/.cpp

### c_Clipmap グループ（1本）
- [x] `ref_vsm_clipmap.md`        … VirtualShadowMapClipmap.h/.cpp

### d_Projection グループ（2本）
- [x] `ref_vsm_projection.md`     … VirtualShadowMapProjection.h/.cpp
- [x] `ref_vsm_shaders.md`        … VirtualShadowMapShaders.h

---

## Phase 3 — コード実行フロー追加・Reference 強化

### Overview
- [x] 04_vsm_overview: Initialize → BeginMarkPages → BuildPageAllocations → RenderNanite/NonNanite → UpdateHZB → Projection → PostRender 全体フロー追加

### Details（`## コード実行フロー` + `## 関連リファレンス` 追加）
- [x] a_vsm_array       : Initialize → BeginMarkPages → BuildPageAllocations → RenderNanite/NonNanite → UpdateHZB → PostRender フロー
- [x] b_vsm_cache_manager: ProcessInvalidations（無効化パイプライン）+ ExtractFrameData（フレーム間データ転送）フロー
- [x] c_vsm_clipmap     : コンストラクタ → AllocateDirectional → BeginMarkPages → AddRenderViews → UpdateCachedFrameData フロー
- [x] d_vsm_projection  : RenderVirtualShadowMapProjection (PerLight / OnePass) → SMRT → CompositeVirtualShadowMapMask フロー

### Reference（`> [!note]-` callout 3本追加）
- [x] ref_vsm_array         : FVirtualShadowMapArray ライフサイクル / ClipmapLevel 符号判別 / Nanite EPipeline::Shadows 統合
- [x] ref_vsm_cache_manager : ISceneExtension 永続性 / ProcessInvalidations GPU 実行 / ExtractFrameData と PooledBuffer
- [x] ref_vsm_clipmap       : GetLevelRadius 指数スケール / CornerRelativeOffset クリップマップ移動 / bIsFirstPersonShadow 専用クリップマップ
- [x] ref_vsm_projection    : SMRT ExtrapolateMaxSlope ペナンブラ推定 / OnePass vs PerLight 使い分け / EVirtualShadowMapProjectionInputType HairStrands
- [x] ref_vsm_shaders       : HLSL 2021 有効化 / DefaultCSGroupXY vs DefaultCSGroupX / 継承ツリー
