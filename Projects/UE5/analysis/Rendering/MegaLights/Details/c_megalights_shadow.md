# c: MegaLights シャドウ統合

- 対象ファイル: `MegaLightsRayTracing.cpp` / `MegaLights.cpp`
- 概要: [[06_megalights_overview]]

---

## 概要

MegaLights のシャドウ計算は **EnabledRT（HW Ray Tracing）** または  
**EnabledVSM（Virtual Shadow Maps）** の2モードで動作する。  
モードはライストごとに `GetMegaLightsMode()` で決定される。

---

## モード選択

```cpp
// MegaLights.h:55
EMegaLightsMode MegaLights::GetMegaLightsMode(
    const FSceneViewFamily& ViewFamily,
    uint8 LightType,
    bool bLightAllowsMegaLights,
    TEnumAsByte<EMegaLightsShadowMethod::Type> ShadowMethod)
{
    // 優先度: Disabled > EnabledRT > EnabledVSM
    if (!IsEnabled(ViewFamily) || !bLightAllowsMegaLights)
        return EMegaLightsMode::Disabled;

    if (UseHardwareRayTracing(ViewFamily) && IsHardwareRayTracingSupported(ViewFamily))
        return EMegaLightsMode::EnabledRT;  // r.MegaLights.HardwareRayTracing=1 時

    return EMegaLightsMode::EnabledVSM;     // HW RT 非対応 or 無効時
}
```

---

## EnabledRT パス（HW Ray Tracing）

```
MegaLights::RayTraceLightSamples()           MegaLights.cpp / MegaLightsRayTracing.cpp
  │
  ├─ サンプルバッファ（LightSamples / LightSampleRays）を入力として受け取る
  │
  ├─ [Inline RT 有効] FMegaLightsInlineRTCS
  │   → TLAS に対してシャドウレイを発射（RayQuery）
  │   → シャドウ結果を LightSamples の Visibility ビットに書き込む
  │
  └─ [Inline RT 非対応] FMegaLightsDispatchRTCS
      → DispatchRays（RGS: Ray Generation Shader）で発射
      → Miss Shader: 遮蔽なし → Visibility = 1.0
      → AnyHit Shader: 遮蔽あり → Visibility = 0.0
```

---

## EnabledVSM パス（Virtual Shadow Maps）

```
MegaLights::MarkVSMPages()
  │
  ├─ LightSamples テクスチャからサンプルされたライスト位置を取得
  ├─ VSM ページマーキング（RenderVirtualShadowMapProjectionMaskBits と連携）
  │   → 必要な VSM ページを「レンダリングが必要」とマーク
  └─ 実際のシャドウ判定はVSM Shadow Map の既存テクスチャを参照
```

---

## シャドウ結果のデータフロー

```
[EnabledRT]
LightSampleRays（方向・距離）
  └─ RayTraceLightSamples()
      └─ TLAS へのレイキャスト
          → Visibility（0/1 または AO 値）
              → LightSamples テクスチャのビットに書き込む

[EnabledVSM]
LightSamples（サンプルされたライストインデックス）
  └─ MarkVSMPages()
      → VirtualShadowMapArray に必要ページをマーク
      → VSMプロジェクションテクスチャから Shadow 値を読む
          → Resolve パスでライスト寄与に乗算
```

---

## 主要 CVar（シャドウ関連）

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.MegaLights.HardwareRayTracing` | 1 | HW RT シャドウ使用（0=VSM のみ）|
| `r.MegaLights.UseInlineRayTracing` | 1 | Inline RT（`RayQuery`）を使用 |
| `r.MegaLights.FarField` | 1 | Far Field GDF（遠距離シャドウ）使用 |

---

## 関連リファレンス

- [[ref_megalights_core]] — `FMegaLightsFrameTemporaries` の VirtualShadowMapArray 参照
- [[a_megalights_pipeline]] — シャドウの呼び出し位置（ステップ C）

---

## EnabledRT / EnabledVSM 条件分岐フロー

```
【呼び出し元の分岐ロジック】
  FMegaLightsViewContext::Setup()
    │
    └─ bInUseVSM = ShadowSceneRenderer.AreAnyLightsUsingMegaLightsVSM()
        ├─ true  → VSM パス有効（MarkVSMPages() / VSM テクスチャ参照）
        └─ false → RT のみ

  GenerateMegaLightsSamples() → GenerateSamples() 後
    │
    ├─ [bUseVSM=true] ViewContext.MarkVSMPages()
    │   → VSM ページマーキング（RenderVirtualShadowMaps() 前に実行必須）
    │
    └─ RenderMegaLights() → ViewContext.RayTrace(ShadingPassIndex)

【EnabledRT パス詳細分岐】
  MegaLights::RayTraceLightSamples()           MegaLightsRayTracing.cpp
    │
    ├─ MegaLights::UseInlineRayTracing() チェック
    │   = r.MegaLights.UseInlineRayTracing=1 && RHI がサポート
    │
    ├─ [Inline RT 有効] FMegaLightsInlineRTCS
    │   │ HLSL: RayQuery<RAY_FLAG_FORCE_OPAQUE> を使用
    │   │
    │   ├─ per sample:
    │   │   RayQuery.TraceRayInline(TLAS, Ray(Origin, Dir, TMin, TMax))
    │   ├─ RayQuery.Proceed() → CommittedStatus チェック
    │   │   COMMITTED_NOTHING      → Visibility = 1.0（非遮蔽）
    │   │   COMMITTED_TRIANGLE_HIT → Visibility = 0.0（遮蔽）
    │   └─ LightSamples テクスチャの Visibility ビットを更新
    │
    └─ [Inline RT 非対応] FMegaLightsDispatchRTCS + RGS/Miss/AnyHit
        ├─ DispatchRays(LightSampleRays テクスチャのサイズで発射)
        ├─ RGS（Ray Generation Shader）: レイをスケジューリング
        ├─ Miss Shader: 遮蔽なし → Visibility = 1.0
        └─ AnyHit Shader: 最初のヒット → Visibility = 0.0 で AcceptHitAndEndSearch

【EnabledVSM パス詳細分岐】
  [GenerateSamples より前のフェーズ]
  MegaLights::MarkVSMPages()                   MegaLights.cpp
    │
    ├─ LightSamples テクスチャからサンプルしたライスト位置を取得
    ├─ ライスト位置 → VSM ページアドレスを計算
    └─ VirtualShadowMapArray.MarkPagesForRendering()
        → 後続の RenderVirtualShadowMaps() でそのページのみレンダリング

  [Resolve フェーズ内]
  FMegaLightsResolveCS（MegaLightsResolve.cpp）:
    SampleVirtualShadowMap(
        VirtualShadowMapSamplingParameters,
        VirtualShadowMapId,
        WorldPosition)
    → VSM テクスチャから Shadow 深度値を取得
    → Visibility = (ShadowDepth > ReceiverDepth) ? 1.0 : 0.0

【FarField GDF（r.MegaLights.FarField=1）】
  EnabledRT の後処理として追加レイを発射:
  → TLAS の FarField レイヤー（粗いジオメトリ近似）に対してレイ発射
  → 近距離では通常 TLAS / 遠距離では FarField TLAS にシームレスに切り替え
  → 遠距離ライスト（屋外シーン等）の遮蔽精度を向上
```
