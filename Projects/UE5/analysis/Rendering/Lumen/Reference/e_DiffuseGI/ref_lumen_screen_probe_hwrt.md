# リファレンス：LumenScreenProbeHardwareRayTracing.cpp

- グループ: e - Diffuse GI
- 上位: [[e_lumen_diffuse_gi]]
- 関連: [[ref_lumen_screen_probe_gather]] | [[ref_lumen_hwrt_common]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenScreenProbeHardwareRayTracing.cpp`

---

## 概要

Screen Probe の **Hardware Ray Tracing（HW RT）** トレース実装ファイル。  
`TraceScreenProbes()` の代わりに `RenderHardwareRayTracingScreenProbe()` が呼ばれ、  
HW RT で各プローブのレイを発射して Radiance をサンプリングする。  
`#if RHI_RAYTRACING` ブロック内にのみ実装される。

---

## エントリポイント

```cpp
void RenderHardwareRayTracingScreenProbe(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FSceneTextureParameters& SceneTextures,
    FScreenProbeParameters& CommonDiffuseParameters,  // 入出力: プローブパラメータ
    const FViewInfo& View,
    const FLumenCardTracingParameters& TracingParameters,
    FLumenIndirectTracingParameters& DiffuseTracingParameters,
    const LumenRadianceCache::FRadianceCacheInterpolationParameters& RadianceCacheParameters,
    ERDGPassFlags ComputePassFlags);
```

---

## HW RT Screen Probe トレースの流れ

```
RenderHardwareRayTracingScreenProbe()
  │
  ├─ [コンパクト化]
  │   CompactTraces() で距離でカリング
  │   → CompactedTraceTexelData に有効なプローブ・方向のリストを構築
  │
  ├─ [HW RT Dispatch]
  │   CompactedTraceTexelData をもとに Indirect Dispatch
  │   ├─ RayGen / Inline RT のどちらか（IsInlineSupported() で選択）
  │   ├─ ペイロード: LumenMinimal（Surface Cache サンプリング）
  │   │   または Hit Lighting（UseHitLighting() = true の場合）
  │   │
  │   ├─ ヒット時: FinalLightingAtlas から Radiance をサンプリング
  │   │   Hit Lighting モードの場合はフルマテリアル評価
  │   └─ ミス時: Radiance Cache から補間 または Skylight
  │
  ├─ [Far Field]
  │   bFarField = true の場合:
  │   → 近距離 TLAS ミス後に Far Field TLAS でもトレース
  │
  └─ TraceRadiance / TraceHit テクスチャに書き込み
        → LumenScreenProbeFiltering.cpp でフィルタリング
```

---

## SW RT との差異

| 比較項目 | SW RT（TraceScreenProbes）| HW RT（RenderHardwareRayTracingScreenProbe）|
|---------|------|------|
| 精度 | SDF 近似誤差あり | ジオメトリに対して正確 |
| フォールバック | HZB → Mesh SDF → Global SDF | HZB スクリーン → HW RT のみ |
| GPU コスト | 低め | 高め（BLAS トラバーサル）|
| Hit Lighting | 不可 | 可（UseHitLighting() = true 時）|

---

## Hit Lighting との連携

```cpp
// HW RT + Hit Lighting モード（r.Lumen.HardwareRayTracing.LightingMode = 1）の場合:
bool LumenScreenProbeGather::UseHitLighting(
    const FViewInfo& View,
    EDiffuseIndirectMethod DiffuseIndirectMethod);

// UseHitLighting = true の場合:
// → ヒット点でフル材質評価 + Direct Lighting の計算
// → Surface Cache には依存しない高品質モード
// → GPU コストが非常に高い（映像制作向け）
```
