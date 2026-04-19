# Rendering ソースマップ作成 タスクチェックリスト

Rendering は Details/Reference/Flow が全て完了済みだが、  
フェーズ 0（ソースマップ）は後から導入された概念のため未作成。  
`ue5-dive` でのアドホック調査効率化のために後追いで作成する。

---

## システムレベル
- [x] `_source_map.md` … Rendering 全体のモジュール→ファイル→クラス対応表（2026-04-19 完了）

## サブフォルダ別（各フォルダ内に `_source_map.md` を作成）

### CPU 側（23 フォルダ）

Stage 1（コアインフラ 5 本、2026-04-19 完了）:
- [x] `SceneRenderer/_source_map.md`
- [x] `RDG/_source_map.md`
- [x] `RHI/_source_map.md`
- [x] `MeshPassProcessor/_source_map.md`
- [x] `GPUScene/_source_map.md`

Stage 2（主要パス 8 本、2026-04-19 完了）:
- [x] `DepthPrepass/_source_map.md`
- [x] `BasePass/_source_map.md`
- [x] `DeferredLighting/_source_map.md`
- [x] `Shadow/_source_map.md`
- [x] `VirtualShadowMaps/_source_map.md`
- [x] `Nanite/_source_map.md`
- [x] `Lumen/_source_map.md`
- [x] `PostProcess/_source_map.md`

Stage 3（特化機能 10 本、2026-04-19 完了）:
- [x] `HZB/_source_map.md`
- [x] `Translucency/_source_map.md`
- [x] `SkyAtmosphere/_source_map.md`
- [x] `Fog/_source_map.md`
- [x] `Decals/_source_map.md`
- [x] `SSAO/_source_map.md`
- [x] `DistanceField/_source_map.md`
- [x] `RayTracing/_source_map.md`
- [x] `Substrate/_source_map.md`
- [x] `MegaLights/_source_map.md`

### GPU 側（17 フォルダ、2026-04-19 完了）
- [x] `GPU/BasePass/_source_map.md`
- [x] `GPU/Decals/_source_map.md`
- [x] `GPU/DeferredLighting/_source_map.md`
- [x] `GPU/DepthPrepass/_source_map.md`
- [x] `GPU/Fog/_source_map.md`
- [x] `GPU/GPUScene/_source_map.md`
- [x] `GPU/HZB/_source_map.md`
- [x] `GPU/MegaLights/_source_map.md`
- [x] `GPU/PostProcess/_source_map.md`
- [x] `GPU/RayTracing/_source_map.md`
- [x] `GPU/SSAO/_source_map.md`
- [x] `GPU/SkyAtmosphere/_source_map.md`
- [x] `GPU/Translucency/_source_map.md`
- [x] `GPU/Lumen/_source_map.md`
- [x] `GPU/Nanite/_source_map.md`
- [x] `GPU/VirtualShadowMaps/_source_map.md`
- [x] `GPU/DistanceField/_source_map.md`

---

## 優先度

ソースマップは `ue5-dive` でアドホック調査する際に最も価値が出る。  
**全て一括作成する必要はなく、実際に調査依頼があったフォルダから順次作成**で構わない。

一括で作りたい場合は Sonnet で量産可能（既存 Reference/Details から情報を抽出するだけなので）。

---

**合計**: システムレベル 1 + CPU 23 + GPU 17 = **41 ファイル**
