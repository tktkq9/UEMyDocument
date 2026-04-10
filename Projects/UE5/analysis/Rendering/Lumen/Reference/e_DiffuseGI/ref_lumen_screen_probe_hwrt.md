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

## RenderHardwareRayTracingScreenProbe

```cpp
void RenderHardwareRayTracingScreenProbe(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FSceneTextureParameters& SceneTextures,
    FScreenProbeParameters& CommonDiffuseParameters,
    const FViewInfo& View,
    const FLumenCardTracingParameters& TracingParameters,
    FLumenIndirectTracingParameters& DiffuseTracingParameters,
    const LumenRadianceCache::FRadianceCacheInterpolationParameters& RadianceCacheParameters,
    ERDGPassFlags ComputePassFlags);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（TLAS / Far Field TLAS アクセスに使用）|
| `SceneTextures` | `const FSceneTextureParameters&` | HZB スクリーントレース用テクスチャ |
| `CommonDiffuseParameters` | `FScreenProbeParameters&` | 入出力プローブパラメータ（TraceRadiance / TraceHit UAV に書き込み）|
| `View` | `const FViewInfo&` | カメラビュー |
| `TracingParameters` | `const FLumenCardTracingParameters&` | Surface Cache アトラス・トレース共通パラメータ |
| `DiffuseTracingParameters` | `FLumenIndirectTracingParameters&` | MaxTraceDistance / SurfaceBias 等の GI トレース設定 |
| `RadianceCacheParameters` | `FRadianceCacheInterpolationParameters` | ミス時に Radiance Cache から補間 |
| `ComputePassFlags` | `ERDGPassFlags` | AsyncCompute / Compute の選択フラグ |

### 内部処理フロー

1. **HZB スクリーントレース（先行実行）**
   ```cpp
   // SW RT と共通の HZB スクリーントレースを先に実行
   // → スクリーン内で解決できるレイを事前に消化
   ScreenTraceScreenProbes(GraphBuilder, View, SceneTextures, CommonDiffuseParameters, ...);
   ```

2. **コンパクト化**
   ```cpp
   // スクリーントレースのミスをコンパクト化
   FCompactedTraceParameters CompactedTraceParameters = LumenScreenProbeGather::CompactTraces(
       GraphBuilder, View, CommonDiffuseParameters, /*bCullByDistance*/ true,
       CardTraceEndDistanceFromCamera, MaxTraceDistance, ...);
   ```

3. **HW RT Dispatch**
   ```cpp
   // CompactedTraceTexelData をもとに Indirect DispatchRays
   // ├─ Inline RT (IsInlineSupported()) または RayGen シェーダー
   // ├─ ペイロード: LumenMinimal（高速 Surface Cache サンプリング）
   // │   または Hit Lighting（UseHitLighting() = true の場合）
   // ├─ ヒット時: FinalLightingAtlas から Radiance をサンプリング
   // └─ ミス時: RadianceCacheParameters から補間、または Skylight
   GraphBuilder.AddPass(RDG_EVENT_NAME("RT ScreenProbe"), PassParameters, ComputePassFlags,
       [RayGenShader, ...](FRHIRayTracingCommandList& RHICmdList) {
           RHICmdList.RayTraceDispatchIndirect(...);
       });
   ```

4. **Far Field トレース（bFarField=true の場合）**
   ```cpp
   // 近距離 TLAS のミス後に Far Field TLAS で 2 段目のトレース
   if (DiffuseTracingParameters.bUseFarField) {
       GraphBuilder.AddPass(RDG_EVENT_NAME("RT ScreenProbe FarField"), ...);
   }
   ```

5. **TraceRadiance / TraceHit への書き込み完了**
   ```cpp
   // → LumenScreenProbeFiltering.cpp でフィルタリング（空間フィルタ → テンポラル → SH 射影）
   ```

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — `RenderLumenScreenProbeGather()` でプラットフォームが HW RT 対応の場合に呼ばれる

---

## UseHitLighting 連携

> [!note]- LumenScreenProbeGather::UseHitLighting — HW RT Hit Lighting の使用判定
>
> ```cpp
> bool LumenScreenProbeGather::UseHitLighting(
>     const FViewInfo& View,
>     EDiffuseIndirectMethod DiffuseIndirectMethod);
> ```
>
> **戻り値**: `bool` — `r.Lumen.HardwareRayTracing.LightingMode = 1` かつ DiffuseIndirectMethod が Lumen の場合に `true`
>
> **UseHitLighting = true の場合の動作**:
> - ヒット点でフル材質評価 + Direct Lighting を計算
> - Surface Cache には依存しない高品質モード
> - GPU コストが非常に高い（映像制作向け）
>
> **使用箇所**: [[ref_lumen_screen_probe_hwrt]] — HW RT トレース前にペイロード種別の選択に使用

---

## SW RT との差異

| 比較項目 | SW RT（TraceScreenProbes）| HW RT（RenderHardwareRayTracingScreenProbe）|
|---------|------|------|
| 精度 | SDF 近似誤差あり | ジオメトリに対して正確 |
| フォールバック | HZB → Mesh SDF → Global SDF | HZB スクリーン → HW RT のみ |
| GPU コスト | 低め | 高め（BLAS トラバーサル）|
| Hit Lighting | 不可 | 可（UseHitLighting() = true 時）|
| Far Field | Global SDF 到達範囲のみ | Far Field TLAS で遠景もカバー |
| 対応条件 | 全プラットフォーム | RT 対応 GPU（RHI_RAYTRACING）のみ |

---

## HW RT Screen Probe トレースの流れ

```
RenderHardwareRayTracingScreenProbe()
  │
  ├─ [HZB スクリーントレース（先行）]
  │   スクリーン内で解決できるレイを事前消化
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
  │   ├─ ヒット時: FinalLightingAtlas から Radiance をサンプリング
  │   └─ ミス時: Radiance Cache から補間 または Skylight
  │
  ├─ [Far Field]
  │   bFarField = true の場合:
  │   → 近距離 TLAS ミス後に Far Field TLAS でもトレース
  │
  └─ TraceRadiance / TraceHit テクスチャに書き込み
        → LumenScreenProbeFiltering.cpp でフィルタリング
```
