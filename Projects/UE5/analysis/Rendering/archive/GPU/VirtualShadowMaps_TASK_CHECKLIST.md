# VSM GPU シェーダードキュメント チェックリスト

## 概要
- [x] `01_vsm_gpu_overview.md` … VSM GPU 全処理実行順 + CPU対応表

---

## VSM GPU シェーダー（グループ別）

### a: Invalidation（2本）
- [x] `a_Invalidation/detail_invalidation.md`  … インスタンス状態更新 + キャッシュ無効化キュー処理
- [x] `a_Invalidation/ref_invalidation.md`     … VirtualShadowMapCacheGPUInvalidation / CacheLoadBalancer エントリポイント

### b: Page Management（2本）
- [x] `b_PageManagement/detail_page_management.md`  … ページフラグ生成・物理ページ割り当て・ページマーキング
- [x] `b_PageManagement/ref_page_management.md`     … PageManagement / PhysicalPageManagement / PageMarking エントリポイント

### c: Draw Commands（2本）
- [x] `c_DrawCommands/detail_draw_commands.md`  … CompactViews + ページ単位 Draw Command 構築
- [x] `c_DrawCommands/ref_draw_commands.md`     … CompactViews / BuildPerPageDrawCommands / ShadowCasterBounds エントリポイント

### d: Projection（2本）
- [x] `d_Projection/detail_projection.md`  … SMRT 投影サンプリング + Shadow Mask 合成
- [x] `d_Projection/ref_projection.md`     … VirtualShadowMapProjection / ProjectionComposite エントリポイント

### e: Misc（2本）
- [x] `e_Misc/detail_misc.md`  … Throttle + Stats + Debug
- [x] `e_Misc/ref_misc.md`     … Throttle / CopyStats / PrintStats / Debug エントリポイント

---

合計: 概要 1 + VSM シェーダー 10 = **11 ファイル**
