# Rendering ソースマップ作成 タスクチェックリスト

Rendering は Details/Reference/Flow が全て完了済みだが、  
フェーズ 0（ソースマップ）は後から導入された概念のため未作成。  
`ue5-dive` でのアドホック調査効率化のために後追いで作成する。

---

## システムレベル
- [ ] `_source_map.md` … Rendering 全体のモジュール→ファイル→クラス対応表

## サブフォルダ別（各フォルダ内に `_source_map.md` を作成）

### CPU 側（23 フォルダ）
- [ ] `BasePass/_source_map.md`
- [ ] `Decals/_source_map.md`
- [ ] `DeferredLighting/_source_map.md`
- [ ] `DepthPrepass/_source_map.md`
- [ ] `DistanceField/_source_map.md`
- [ ] `Fog/_source_map.md`
- [ ] `GPUScene/_source_map.md`
- [ ] `HZB/_source_map.md`
- [ ] `Lumen/_source_map.md`
- [ ] `MegaLights/_source_map.md`
- [ ] `MeshPassProcessor/_source_map.md`
- [ ] `Nanite/_source_map.md`
- [ ] `PostProcess/_source_map.md`
- [ ] `RayTracing/_source_map.md`
- [ ] `RDG/_source_map.md`
- [ ] `RHI/_source_map.md`
- [ ] `SceneRenderer/_source_map.md`
- [ ] `Shadow/_source_map.md`
- [ ] `SkyAtmosphere/_source_map.md`
- [ ] `SSAO/_source_map.md`
- [ ] `Substrate/_source_map.md`
- [ ] `Translucency/_source_map.md`
- [ ] `VirtualShadowMaps/_source_map.md`

### GPU 側（17 フォルダ）
- [ ] `GPU/BasePass/_source_map.md`
- [ ] `GPU/Decals/_source_map.md`
- [ ] `GPU/DeferredLighting/_source_map.md`
- [ ] `GPU/DepthPrepass/_source_map.md`
- [ ] `GPU/Fog/_source_map.md`
- [ ] `GPU/GPUScene/_source_map.md`
- [ ] `GPU/HZB/_source_map.md`
- [ ] `GPU/MegaLights/_source_map.md`
- [ ] `GPU/PostProcess/_source_map.md`
- [ ] `GPU/RayTracing/_source_map.md`
- [ ] `GPU/SSAO/_source_map.md`
- [ ] `GPU/SkyAtmosphere/_source_map.md`
- [ ] `GPU/Translucency/_source_map.md`
- [ ] `GPU/Lumen/_source_map.md`
- [ ] `GPU/Nanite/_source_map.md`
- [ ] `GPU/VirtualShadowMaps/_source_map.md`
- [ ] `GPU/DistanceField/_source_map.md`

---

## 優先度

ソースマップは `ue5-dive` でアドホック調査する際に最も価値が出る。  
**全て一括作成する必要はなく、実際に調査依頼があったフォルダから順次作成**で構わない。

一括で作りたい場合は Sonnet で量産可能（既存 Reference/Details から情報を抽出するだけなので）。

---

**合計**: システムレベル 1 + CPU 23 + GPU 17 = **41 ファイル**
