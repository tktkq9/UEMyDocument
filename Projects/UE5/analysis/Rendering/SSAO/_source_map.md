# SSAO ソースマップ

- 対象: GTAO / 旧 SSAO / モバイル AO（Lumen 有効時はスキップ）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[22_ssao_overview]]

通常は GTAO（Ground Truth AO）を使用。HorizonSearch → Integrate → SpatialFilter → TemporalFilter の 4 パスを
Async Compute で実行し、Lumen 有効時は Lumen Short Range AO に置き換わる。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/` |
| メイン | `PostProcess/PostProcessAmbientOcclusion.h/.cpp` |
| モバイル | `PostProcess/PostProcessAmbientOcclusionMobile.cpp` |
| 合成オーケストレーション | `CompositionLighting/CompositionLighting.h/.cpp` |
| シェーダー | `Engine/Shaders/Private/PostProcessAmbientOcclusion.usf`, `GTAO.usf` |

---

## ファイル → クラス対応

### GTAO / SSAO 本体

| ファイル | 主要クラス / 関数 / enum | 役割 | 参照 |
|---------|----------------------|------|------|
| `PostProcessAmbientOcclusion.h/.cpp` | `FGTAOContext`, `FSSAOHelper`, `ESSAOType`, `EGTAOType`, `EGTAOPass`, `AddAmbientOcclusionPasses()` | GTAO/SSAO の各パス登録 | [[Reference/ref_ssao]] |
| `PostProcessAmbientOcclusion.cpp` | HorizonSearch CS / Integrate CS / SpatialFilter CS / TemporalFilter CS | 4 パスシェーダー実装 | 同 |

### オーケストレーション

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `CompositionLighting.h/.cpp` | `FCompositionLighting::ProcessAfterOcclusion()`, `AddAmbientOcclusionPasses()` | AO パスの登録オーケストレーター | [[Reference/ref_composition_lighting]] |

### モバイル

| ファイル | 主要関数 | 役割 |
|---------|--------|------|
| `PostProcessAmbientOcclusionMobile.cpp` | モバイル版 AO | モバイル簡易 SSAO |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()（GBuffer 完了後）
  │
  └─ FCompositionLighting::ProcessAfterOcclusion      CompositionLighting.cpp
       │
       ├─[Lumen 有効]
       │   Lumen Short Range AO（ScreenProbeGather 経由）
       │   以降の GTAO/SSAO はスキップ
       │
       └─[GTAO] FGTAOContext ctx(GTAOType)
           AddAmbientOcclusionPasses                  PostProcessAmbientOcclusion.cpp
             │
             ├─[A] HorizonSearch CS    入力: HZB/SceneDepth + GBufferA 法線
             │                          出力: HorizonAngleTexture
             │
             ├─[B] Integrate CS         HorizonAngle → 立体角積分 → BentNormal + AO
             │
             ├─[C] SpatialFilter CS     深度+法線キーのバイラテラルブラー
             │
             └─[D] TemporalFilter CS    Velocity で再投影 + Neighborhood Clamp
                                        → AO テクスチャ（R8 or R8G8）
       │
       → DeferredLightingComposite で間接光に乗算
```

---

## 主要 enum

### EGTAOType

| enum | 意味 |
|------|------|
| `EOff` | GTAO 無効（SSAO 使用） |
| `EAsyncCombinedSpatial` | HorizonSearch+Integrate を Async、Temporal/Upsample は GFX |
| `EAsyncHorizonSearch` | HorizonSearch のみ Async |
| `ENonAsync` | 全 GFX パイプ |

### EGTAOPass

| flag | 内容 |
|------|------|
| `HorizonSearch` | 水平線角度探索 |
| `HorizonSearchIntegrate` | 探索+積分 Combined |
| `Integrate` | 立体角積分 |
| `SpatialFilter` | 空間フィルタ |
| `TemporalFilter` | 時間フィルタ |
| `Upsample` | ハーフ→フル解像度 |

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.AmbientOcclusion.Method` | `PostProcessAmbientOcclusion.cpp`（0=SSAO, 1=GTAO） |
| `r.GTAO.Quality` | 同 |
| `r.GTAO.HalfRes` | 同 |
| `r.GTAO.UseNormals` | 同 |
| `r.GTAO.TemporalFilter` | 同 |
| `r.AmbientOcclusionStaticFraction` | 同 |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_ssao]] | FGTAOContext / FSSAOHelper |
| Reference | [[Reference/ref_composition_lighting]] | CompositionLighting / AddAmbientOcclusionPasses |

---

## ue5-dive 起点

- 「AO 方式切替」 → `r.AmbientOcclusion.Method` + `EGTAOType`
- 「GTAO パスフロー」 → `PostProcessAmbientOcclusion.cpp:AddAmbientOcclusionPasses`
- 「Async Compute スケジューリング」 → `EGTAOType::EAsyncCombinedSpatial / EAsyncHorizonSearch`
- 「Lumen 時のスキップ」 → `CompositionLighting.cpp:ProcessAfterOcclusion`（Lumen 分岐）
- 「BentNormal」 → Integrate CS で HorizonAngle → BentNormal を同時出力
