# Lumen Hardware Ray Tracing（DXR/RT Core 経由のレイトレース）

- 上位: [[_algorithm_index]]
- 関連: [[lumen_sw_rt]] / [[lumen_surface_cache]] / [[lumen_radiance_cache]] / [[lumen_final_gather]]
- 採用システム: Lumen GI / Reflections / Translucency / Visualize の HW トレース経路（Inline RT または RayGen）, ShortRangeAO HW
- 出典 ID: **S10**（[[_source_index]]）— §6 Hardware Ray Tracing
- 関連仕様: DirectX Raytracing 1.1（Inline RT / TraceRayInline）

---

## 1. 何のためのアルゴリズムか

DXR / Vulkan RT 拡張の **TraceRay / TraceRayInline** で **三角形 BLAS** に対する正確なレイヒット判定を実行。SW RT (Mesh SDF) より高品質だが、BLAS 構築コスト + 動的シーンの更新負荷が高い。

### SW RT に対する優位

- **三角形精度**: SDF 由来の細部欠損・leak が無い
- **薄い壁・細い枝**が正しくヒット
- **マスク付きマテリアル**を Any-Hit Shader で正確に処理
- **Hit Lighting モード**で BasePass 同等の material 評価が可能（Surface Cache 不依存）

### 課題

- BLAS 構築・更新が CPU/GPU 双方で重い（特に Skinned mesh）
- "Note: Hardware ray tracing has significant scene update costs for scenes with more than 100k instances" (UE 公式 CVar コメント)
- Nanite 互換のため **Nanite Ray Tracing**（プロキシメッシュ生成）が必要

---

## 2. 理論

### 2.1 RT パイプラインの選択肢

UE では 2 つを使い分け:

| 種類 | 仕組み | 用途 |
|-----|------|------|
| **Inline RT (TraceRayInline)** | compute / pixel shader 内から直接レイ射出、shader pipeline 切替なし | Lumen の主要トレース（Surface Cache モード） |
| **RayGen (TraceRay)** | 専用 RayGen / ClosestHit / AnyHit / MissShader | Hit Lighting モード、Visualize、Translucency |

`r.Lumen.HardwareRayTracing.Inline = 1` で Inline 優先（Inline 不可なら RayGen フォールバック）。

### 2.2 LightingMode（3 モード）

```
0 - Surface Cache: ヒット点で Surface Cache サンプリングのみ。最速。
1 - HitLighting: GI/Reflections 全てで full material + shadow ray。最重・最高品質。
2 - HitLightingForReflections: Reflections だけ HitLighting、GI は SurfaceCache。
```

### 2.3 Self-Intersection 回避

BLAS と raster の頂点が完全一致しない場合、自分自身を誤ってヒットする問題:

```
0 - Disabled
1 - Retrace: 最初の backface ヒットを SkipBackFaceHitDistance までスキップして再トレース
2 - AHS: AnyHitShader で全 backface ヒットをスキップ
3 - Auto: プラットフォーム依存で 1 or 2 を選択（Inlined Callbacks 対応 GPU は AHS）
```

### 2.4 SER (Shader Execution Reordering)

NVIDIA Ada (RTX 40 系) 以降の機能。レイヒット後の material 評価で **同一 material のスレッドを再グループ化** → divergence 緩和。

`r.Lumen.HardwareRayTracing.ShaderExecutionReordering = 1`（デフォルト）。

### 2.5 Far Field Bias

スクリーン距離超え（Far Field、典型 200 m 超）のレイは **別 BLAS（粗いプロキシ）** に切替。`FarFieldBias = 200` がレイ origin 側のオフセット。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `LumenHardwareRayTracingCommon.cpp/.h` | HW RT 共通設定、CVar、HitLighting モード判定 |
| `LumenHardwareRayTracingMaterials.cpp` | Hit Lighting 用 material shader binding |
| `LumenScreenProbeHardwareRayTracing.cpp` | ScreenProbe HW RT pass |
| `LumenReflectionHardwareRayTracing.cpp` | Reflection HW RT pass |
| `LumenRadianceCacheHardwareRayTracing.cpp` | Radiance Cache 更新 HW RT pass |
| `LumenSceneDirectLightingHardwareRayTracing.cpp` | Direct Lighting シャドウ HW RT |
| `LumenTranslucencyVolumeHardwareRayTracing.cpp` | Translucency Volume 更新 |
| `LumenVisualizeHardwareRayTracing.cpp` | デバッグ visualize |
| `LumenShortRangeAOHardwareRayTracing.cpp` | 接触陰用 short-range AO |

### 3.2 HW RT 有効判定（`LumenHardwareRayTracingCommon.cpp:175-185`）

```cpp
bool Lumen::UseHardwareRayTracing(const FSceneViewFamily& ViewFamily)
{
#if RHI_RAYTRACING
    return IsRayTracingEnabled(ViewFamily.GetShaderPlatform())
        && (LumenHardwareRayTracing::IsInlineSupported() || LumenHardwareRayTracing::IsRayGenSupported())
        && CVarLumenUseHardwareRayTracing.GetValueOnAnyThread() != 0
        && ViewFamily.Views[0]->IsRayTracingAllowedForView();
#else
    return false;
#endif
}
```

`IsInlineSupported()` は `GRHISupportsInlineRayTracing`、`IsRayGenSupported()` は `GRHISupportsRayTracingShaders && GRHISupportsRayTracingDispatchIndirect`。

### 3.3 HitLighting モード判定（`LumenHardwareRayTracingCommon.cpp:247-282`）

```cpp
LumenHardwareRayTracing::EHitLightingMode LumenHardwareRayTracing::GetHitLightingMode(...)
{
    if (!LumenHardwareRayTracing::IsRayGenSupported()) {
        return EHitLightingMode::SurfaceCache;  // RayGen 必須
    }
    if (DiffuseIndirectMethod != EDiffuseIndirectMethod::Lumen) {
        return EHitLightingMode::HitLightingForReflections;  // Standalone Reflections は強制
    }
    int32 LightingModeInt = CVarLumenHardwareRayTracingLightingMode.GetValueOnAnyThread();
    // PostProcessVolume の override も反映...
    return (EHitLightingMode)FMath::Clamp(LightingModeInt, 0, (int)EHitLightingMode::MAX - 1);
}
```

### 3.4 Self-Intersection 回避モード選択（`LumenHardwareRayTracingCommon.cpp:198-210`）

```cpp
LumenHardwareRayTracing::EAvoidSelfIntersectionsMode LumenHardwareRayTracing::GetAvoidSelfIntersectionsMode()
{
    int32 Mode = CVarLumenRadiosityHardwareRayTracingAvoidSelfIntersections.GetValueOnAnyThread();
    if (Mode == 3) {  // Auto
        return GRHIGlobals.RayTracing.SupportsInlinedCallbacks
            ? EAvoidSelfIntersectionsMode::AHS
            : EAvoidSelfIntersectionsMode::Retrace;
    }
    return (EAvoidSelfIntersectionsMode)FMath::Clamp(Mode, 0, ...);
}
```

### 3.5 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.Lumen.HardwareRayTracing` | 1 | HW RT マスタースイッチ |
| `r.Lumen.HardwareRayTracing.Inline` | 1 | Inline RT 優先（0 で RayGen） |
| `r.Lumen.HardwareRayTracing.LightingMode` | 0 | 0=SurfaceCache / 1=HitLighting / 2=HitLightingForRefl |
| `r.Lumen.HardwareRayTracing.HitLighting.DirectLighting` | 1 | HitLighting 時に shadow ray 評価 |
| `r.Lumen.HardwareRayTracing.HitLighting.ShadowMode` | SOFT | 0=OFF / 1=Hard / 2=Area |
| `r.Lumen.HardwareRayTracing.HitLighting.Skylight` | 2 | 0/1/2（Standalone Reflections のみ） |
| `r.Lumen.HardwareRayTracing.HitLighting.ReflectionCaptures` | 0 | Reflection Capture を HitLighting で適用 |
| `r.Lumen.HardwareRayTracing.HitLighting.ForceOpaque` | false | AHS スキップで高速化（マスク不正確） |
| `r.Lumen.HardwareRayTracing.ShaderExecutionReordering` | true | SER ON/OFF |
| `r.Lumen.HardwareRayTracing.AvoidSelfIntersections` | 3 | 0/1/2/3（Disabled/Retrace/AHS/Auto） |
| `r.Lumen.HardwareRayTracing.PullbackBias` | 8.0 | Screen Trace から HW RT に切替時のレイ origin pullback |
| `r.Lumen.HardwareRayTracing.FarFieldBias` | 200.0 | Far Field トレース origin offset |
| `r.Lumen.HardwareRayTracing.MaxIterations` | 8192 | BVH トラバーサル最大ステップ |
| `r.Lumen.HardwareRayTracing.MeshSectionVisibilityTest` | 1 | mesh section 単位の可視性テスト |
| `r.Lumen.HardwareRayTracing.MinTraceDistanceToSampleSurfaceCache` | 10.0 | feedback loop 防止用最小距離 |
| `r.Lumen.HardwareRayTracing.SurfaceCacheSampling.DepthBias` | 10.0 | カード→ヒット投影の深度バイアス |
| `r.Lumen.HardwareRayTracing.SurfaceCacheAlphaMasking` | 0 | Surface Cache の Opacity でアルファマスク（重い） |

### 3.6 HitLighting 時の `FRayTracingSceneOptions`（`LumenHardwareRayTracingCommon.cpp:232-245`）

```cpp
void LumenHardwareRayTracing::SetRayTracingSceneOptions(...)
{
    if (ReflectionsMethod == EReflectionsMethod::Lumen
        && LumenReflections::UseHitLighting(View, DiffuseIndirectMethod) 
        && LumenReflections::UseTranslucentRayTracing(View))
    {
        SceneOptions.bTranslucentGeometry = true;
    }
    if (RayTracedTranslucency::IsEnabled(View))
    {
        SceneOptions.bTranslucentGeometry = true;
    }
}
```

→ HitLighting 時に半透明ジオメトリも BLAS に含める。

---

## 4. 近似・省略の差分

| 項目 | フル Path Tracing | Lumen HW RT (SurfaceCache) | Lumen HW RT (HitLighting) |
|------|---------------|-------------------------|------------------------|
| ヒット時評価 | 完全 material | Surface Cache サンプリング | 完全 material 再評価 |
| Shadow Ray | 必要 | 不要（Surface Cache の direct） | r.Lumen...DirectLighting=1 で発射 |
| Skylight | unbiased | 不可 / 1 / Standalone のみ | フル評価可 |
| AnyHit | 全マテリアル | 必要なら | 必要なら |
| 多重バウンス | 任意回 | 1 hop（Surface Cache 内に Radiosity 多重バウンス） | 1 hop（GI は Surface Cache フォールバック） |
| GPU コスト | 巨大 | 軽 | 中〜重 |

**ポイント**: Lumen は **多重バウンスを Surface Cache 上の Radiosity に押し付けることで、レイトレースは 1 hop に抑えている** のが核心設計。

---

## 5. パラメータと CVar

§3.5 にまとめ済み。

---

## 6. 代替手法との比較

| 手法 | BLAS 三角形精度 | Hit 評価 | コスト | UE 採用 |
|------|------------|--------|------|--------|
| **Lumen HW RT (SurfaceCache)** | ◎ | アトラス sample | ★★ | **HW RT 標準** |
| **Lumen HW RT (HitLighting)** | ◎ | フル material | ★★★★ | 高品質モード |
| Lumen SW RT (MeshSDF) | × (SDF) | アトラス sample | ★★ | [[lumen_sw_rt]] |
| 古典 Path Tracer | ◎ | フル material 多重 | ★★★★★ | UE Path Tracer モード |
| RTXDI / ReSTIR DI | ◎ | フル + 重要度サンプリング | ★★★ | NVIDIA 系（UE 統合は限定） |

### Inline RT vs RayGen の選択

- **Inline RT**: shader pipeline state を切らず compute から直接 trace、CPU 側 dispatch コスト低、SER 適用可
- **RayGen**: 専用 RayGen/CHS/AHS/Miss shader で柔軟、HitLighting や Visualize で必要
- Lumen は **Inline 優先 + RayGen 必要時のみ切替** のハイブリッド

---

## 7. 参考資料

- [S10] Wright / Heitz / Hillaire 2022 §6 — HW RT 設計、HitLighting、Far Field
- DirectX Raytracing 1.1 (Inline RT) 仕様
- NVIDIA "Shader Execution Reordering" Whitepaper（Ada 世代）
- 関連: [[lumen_sw_rt]] / [[lumen_surface_cache]]

---

## 8. 相談用フック

- **理解度チェック**:
  - SurfaceCache と HitLighting の違い → §2.2、ヒット時の評価方法
  - なぜ Lumen は 1 hop に抑える？ → §4、Surface Cache + Radiosity に多重バウンスを押し付ける設計
  - Inline RT と RayGen の使い分け → §6
- **コード深掘り候補**:
  - `LumenHardwareRayTracingMaterials.cpp` の HitGroup 生成（Closest Hit Shader バインディング）
  - SER 有効時の パス分岐（`UseShaderExecutionReordering()`）
  - Self-Intersection AHS モードのシェーダ実装
- **未読箇所**:
  - S10 §6.4 Far Field の prox mesh 生成
  - Translucency RayTracing と Hit Lighting の相互作用（`RayTracedTranslucency`）
- **次の派生**:
  - Hit Lighting Lighting Grid → [[lumen_hit_lighting_grid]]（未着手）
  - Nanite Ray Tracing プロキシ → [[../Geometry/...]]
  - ScreenProbe HW RT 詳細パス → [[lumen_final_gather]]
