# c: Reflection / SkyLight

- 対象ファイル: `DeferredShadingRenderer.cpp` / `ReflectionEnvironmentCapture.h/.cpp`
- 概要: [[16_deferred_lighting_overview]]

---

## 概要

`RenderDeferredReflectionsAndSkyLighting()` は **鏡面反射・スカイライトの寄与を SceneColor に合成するパス**。  
Lumen Reflections が有効な場合は Lumen の結果を利用し、  
無効な場合は Reflection Capture（キューブマップ）または SSR（Screen Space Reflection）を使用する。

---

## 処理の種別

| 処理 | 条件 | 内容 |
|-----|------|------|
| Lumen Reflections | `ShouldRenderLumenReflections()` | Lumen で計算済みの反射テクスチャを合成 |
| Screen Space Reflection | `r.SSR.Quality > 0` | 前フレームの SceneColor を利用した SS 反射 |
| Reflection Capture | 常時（フォールバック）| 事前キャプチャしたキューブマップ（IBL）|
| SkyLight | `ShouldRenderSkyLight()` | Sky Light コンポーネントの環境光 |

---

## フロー概略

```
RenderDeferredReflectionsAndSkyLighting()       DeferredShadingRenderer.cpp:3339
  │
  ├─ [ShouldRenderLumenReflections()]
  │   └─ Lumen の反射テクスチャを既に保持 → FinalComposite で合成済み
  │       （このパスでは追加作業なし）
  │
  ├─ [SSR有効 && !Lumen]
  │   AddScreenSpaceReflectionPasses()
  │   → HZB を使った Cone-Traced SSR
  │   → TemporalAccumulate（SSR ノイズ削減）
  │
  ├─ ReflectionEnvironment:
  │   RenderDeferredReflections()
  │   → Reflection Capture キューブマップのブレンド
  │   → BRDF LUT + IBL 評価
  │   → SceneColor に加算
  │
  └─ SkyLight:
      RenderSkyLightForDeferred()
      → ConvolvedSkyLight テクスチャ（ディフューズ + スペキュラー）
      → AO と組み合わせて SceneColor に加算
```

---

## Reflection Capture の種類

```cpp
// ReflectionCaptureComponent.h
// - USphereReflectionCaptureComponent  : 球形キャプチャ範囲
// - UBoxReflectionCaptureComponent     : ボックス形キャプチャ範囲（室内用）
// - UPlanarReflectionComponent         : 平面反射（水面等）

// シェーダー側の読み取り
// ReflectionEnvironmentShared.ush
// GetOffSpecularPeakReflectionDir() → BRDF ピーク反射方向
// GetSkyLightReflection()           → ConvolvedSkyLight テクスチャサンプル
```

---

## Lumen Reflections との関係

```
[Lumen 有効時]
RenderLumenReflections()           ← AsyncCompute で GBuffer と並列実行
  └─ ReflectionTexture を生成

RenderDeferredReflectionsAndAmbientOcclusion()
  └─ DiffuseIndirectComposite.usf で Lumen 反射テクスチャを SceneColor に合成
      ※ このパスでは Lumen 反射を使わない

[Lumen 無効時]
RenderDeferredReflectionsAndSkyLighting()
  └─ Reflection Capture / SSR を使って反射を計算
```

---

## 関連リファレンス

- [[ref_light_rendering]] — `FDeferredLightPS` 内の Sky Light 統合
- [[ref_light_params]] — `FReflectionUniformParameters`

---

## ReflectionCapture → SkyLight → Lumen Reflection 合成 詳細フロー

```
【判定・ルート選択】
FDeferredShadingSceneRenderer::RenderDeferredReflectionsAndSkyLighting()
  │
  ├─ ShouldRenderLumenReflections()
  │   = LumenSettings.bEnabled && SupportLumenGI && ...
  │   → true: Lumen ルートへ
  │
  ├─ IsReflectionEnvironmentAvailable()
  │   = ReflectionCapture が1個以上存在する
  │
  └─ bScreenSpaceReflections = r.SSR.Quality > 0 && !Lumen

──────────────────────────────────────────────────────────
【Lumen 有効ルート】

[AsyncCompute で GBuffer と並列]
RenderLumenReflections()
  → Lumen::RenderReflections() → ReflectionTexture（R11G11B10F）を生成

[RenderDeferredReflectionsAndSkyLighting の中]
DiffuseIndirectComposite.usf:
  └─ Lumen ReflectionTexture を読み込み
  └─ Roughness に応じて Reflection Capture と Lumen 結果をブレンド
  └─ SceneColor に加算

──────────────────────────────────────────────────────────
【Lumen 無効ルート】

[SSR パス]
AddScreenSpaceReflectionPasses()
  ├─ HZB をトレースして前フレーム SceneColor からヒット色を取得
  └─ TemporalAccumulate で SSR ノイズを削減
  → SSRTexture（R11G11B10F）生成

[Reflection Capture パス]
RenderDeferredReflections()
  ├─ 各 ReflectionCapture の影響範囲（球/ボックス）をブレンド
  │   → 近いキャプチャほど高ウェイト（距離フェード）
  ├─ GBuffer.Roughness から Mip Level を決定
  │   MipLevel = Roughness² × MaxMipLevel
  ├─ ConvolvedCubemap.SampleLevel(Dir, MipLevel) でサンプル
  ├─ PreIntegratedGF テクスチャ（BRDF LUT）で Specular を評価
  └─ SSRTexture とブレンド（SSR 寄与エリアは Capture を抑制）
  → SceneColor に加算

[SkyLight パス]
RenderSkyLightForDeferred()
  ├─ ConvolvedSkyLight テクスチャ（ディフューズ用 SH + スペキュラー用 Cubemap）
  ├─ SSAO / DFAO で遮蔽マスクを適用
  └─ SceneColor に加算

──────────────────────────────────────────────────────────
【最終合成での Lumen/非 Lumen ブレンド（共通）】
  DiffuseIndirectComposite.usf
    IndirectDiffuse = Lumen IrradianceTexture (or SH)
    IndirectSpecular = Lumen Reflection (or SSR + Capture ブレンド)
    AmbientOcclusion = SSAO * DFAO
    SceneColor += IndirectDiffuse * AO * BaseColor + IndirectSpecular
```

> [!note]- Reflection Capture の更新タイミング
> 静的 Reflection Capture はレベルビルド時に事前キャプチャ。
> `r.ReflectionCaptureResolution`（デフォルト 128）のキューブマップを圧縮して保持する。
> ランタイム更新は `r.AlwaysResolveRTLC=1` または `UReflectionCaptureComponent::MarkDirty()` で行う。

> [!note]- SSR と Lumen のブレンド境界
> Lumen 有効時でも `Roughness > r.Lumen.Reflections.MaxRoughness`（デフォルト 0.4）の
> 粗いマテリアルは Reflection Capture にフォールバックする。
> 境界付近ではブレンドされるため不連続なアーティファクトは発生しない。
