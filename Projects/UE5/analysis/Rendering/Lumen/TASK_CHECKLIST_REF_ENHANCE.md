# Lumen Reference 拡張タスクリスト

**方針**: 各 Reference ファイルの関数ブロックに「パラメータ表 / 呼び出し元 / 内部動作」を追加（案A）  
**作業開始日**: 2026-04-10  
**優先度**: グループ a → f の順

---

## Common（3件）

- [ ] `Common/ref_lumen_core.md` — LumenScene.h の基本型・定数・有効化関数
- [ ] `Common/ref_lumen_view_state.md` — FLumenViewState, FLumenSceneFrameTemporaries
- [ ] `Common/ref_lumen_visualize.md` — LumenVisualize.h のデバッグ可視化関数

---

## a: SurfaceCache（7件）

- [x] `a_SurfaceCache/ref_lumen_scene_data.md` — FLumenSceneData, FLumenCardData
- [x] `a_SurfaceCache/ref_lumen_mesh_cards.md` — FLumenMeshCards, FLumenCard, FLumenPrimitiveGroup
- [x] `a_SurfaceCache/ref_lumen_surface_cache.md` — UpdateLumenSurfaceCache() 等
- [x] `a_SurfaceCache/ref_lumen_surface_cache_feedback.md` — Surface Cache フィードバックループ
- [x] `a_SurfaceCache/ref_lumen_scene.md` — LumenScene.cpp の更新関数群
- [x] `a_SurfaceCache/ref_lumen_scene_gpu_driven_update.md` — GPU Driven 更新パス
- [x] `a_SurfaceCache/ref_lumen_utils.md` — LumenUtils.h のユーティリティ関数

---

## b: SceneLighting（6件）

- [ ] `b_SceneLighting/ref_lumen_scene_lighting.md` — RenderLumenSceneLighting() 等
- [ ] `b_SceneLighting/ref_lumen_scene_card_capture.md` — カードキャプチャパス
- [ ] `b_SceneLighting/ref_lumen_scene_direct_lighting.md` — 直接光注入パス
- [ ] `b_SceneLighting/ref_lumen_scene_direct_lighting_hwrt.md` — HW Ray Tracing 版直接光
- [ ] `b_SceneLighting/ref_lumen_radiosity.md` — Radiosity（間接光伝播）パス
- [ ] `b_SceneLighting/ref_lumen_scene_rendering.md` — LumenSceneRendering.h の汎用関数

---

## c: Tracing（6件）

- [ ] `c_Tracing/ref_lumen_tracing_utils.md` — FLumenCardTraceParameters, トレースユーティリティ
- [ ] `c_Tracing/ref_lumen_mesh_sdf_culling.md` — SDF カリング関数
- [ ] `c_Tracing/ref_lumen_hwrt_common.md` — HW Ray Tracing 共通設定
- [ ] `c_Tracing/ref_lumen_hwrt_materials.md` — HW Ray Tracing マテリアル処理
- [ ] `c_Tracing/ref_lumen_heightfields.md` — ハイトフィールドトレース
- [ ] `c_Tracing/ref_lumen_irradiance_field.md` — Irradiance Field 構造体・関数

---

## d: RadianceCache（5件）

- [ ] `d_RadianceCache/ref_lumen_radiance_cache.md` — FLumenRadianceCache, 更新エントリポイント
- [ ] `d_RadianceCache/ref_lumen_radiance_cache_internal.md` — キャッシュ内部構造・Probe 管理
- [ ] `d_RadianceCache/ref_lumen_radiance_cache_hwrt.md` — HW RT 版 Radiance Cache
- [ ] `d_RadianceCache/ref_lumen_translucency_radiance_cache.md` — 半透明用 Radiance Cache
- [ ] `d_RadianceCache/ref_lumen_translucency_volume.md` — Translucency Volume 構造体・更新

---

## e: DiffuseGI（7件）

- [ ] `e_DiffuseGI/ref_lumen_diffuse_indirect.md` — RenderLumenDiffuseIndirect(), ShouldRenderLumenDiffuseGI()
- [ ] `e_DiffuseGI/ref_lumen_screen_probe_gather.md` — Screen Probe 収集パス
- [ ] `e_DiffuseGI/ref_lumen_screen_probe_tracing.md` — Screen Probe トレースパス
- [ ] `e_DiffuseGI/ref_lumen_screen_probe_filtering.md` — Screen Probe フィルタリング
- [ ] `e_DiffuseGI/ref_lumen_screen_probe_importance.md` — Importance Sampling 関数
- [ ] `e_DiffuseGI/ref_lumen_screen_probe_hwrt.md` — HW RT 版 Screen Probe
- [ ] `e_DiffuseGI/ref_lumen_short_range_ao.md` — Short Range AO パス

---

## f: Reflections（6件）

- [ ] `f_Reflections/ref_lumen_reflections.md` — RenderLumenReflections(), ShouldRenderLumenReflections()
- [ ] `f_Reflections/ref_lumen_reflection_tracing.md` — 反射トレースパス
- [ ] `f_Reflections/ref_lumen_reflection_hwrt.md` — HW RT 版反射
- [ ] `f_Reflections/ref_lumen_restir.md` — ReSTIR GI 関数群
- [ ] `f_Reflections/ref_lumen_front_layer.md` — Front Layer Translucency
- [ ] `f_Reflections/ref_ray_traced_translucency.md` — Ray Traced Translucency

---

## 追記要件（全ファイル共通）

1. **全関数を網羅** — 主要でない関数も含め、`<details>` 折りたたみで収録
2. **メンバ変数の説明** — 各クラスに `### メンバ変数` テーブルを追加
3. **使用箇所とリンク** — どのクラス・関数から使われるかを記載し、対象 Reference への Obsidian リンク `[[ref_xxx]]` を付与
4. **内部処理フロー** — 処理量の多い関数はステップごとにコード概要を添付

---

## 追記フォーマット

### クラスブロック

```markdown
## ClassName

> **概要**: クラスの1行説明

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| MemberA | `Type` | 説明 |

### 使用箇所
- [[ref_xxx]] — `OtherClass::Function()` でXX目的に参照される
```

---

### 主要関数ブロック

```markdown
## FunctionName

```cpp
ReturnType FunctionName(
    ParamType1 Param1,
    ParamType2 Param2);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| Param1 | `ParamType1` | 説明 |

### 戻り値
`ReturnType` — 説明

### 使用箇所
- [[ref_xxx]] — `CallerFunction()` 内でXX目的に呼ばれる

### 内部処理フロー

1. **ステップ名** — 説明
   ```cpp
   // コード概要
   if (Condition) { DoSomething(); }
   ```
2. **ステップ名** — 説明
   ```cpp
   // コード概要
   ```
```

---

### マイナー関数ブロック（折りたたみ）

```markdown
> [!note]- MinorFunctionName — 短い説明
> 
> ```cpp
> ReturnType MinorFunctionName(ParamType Param);
> ```
> 
> **パラメータ**
> 
> | 引数 | 型 | 説明 |
> |------|-----|------|
> | Param | `ParamType` | 説明 |
> 
> **使用箇所**: [[ref_xxx]] — `CallerFunction()` から呼ばれる
```

---

**合計**: 40ファイル  
**完了数**: 7 / 40（a グループ完了 2026-04-10）
