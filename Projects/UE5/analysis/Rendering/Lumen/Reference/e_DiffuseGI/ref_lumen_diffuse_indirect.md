# リファレンス：LumenDiffuseIndirect.cpp

- グループ: e - Diffuse GI
- 上位: [[e_lumen_diffuse_gi]]
- 関連: [[ref_lumen_screen_probe_gather]] | [[ref_lumen_tracing_utils]] | [[ref_lumen_mesh_sdf_culling]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenDiffuseIndirect.cpp`

---

## 概要

Lumen Diffuse Indirect（拡散間接照明）の**全体オーケストレーション**を担うファイル。  
Screen Probe Gather のカリング設定・フラスタムグリッドのセットアップ・AsyncCompute の切り替えを管理し、  
`RenderLumenFinalGather()` を呼ぶエントリポイントを提供する。

---

## FLumenGatherCvarState

主要 CVar の現在値をまとめた状態管理構造体。  
ランタイム変更を検出してシーンを再構築するために使われる。

```cpp
struct FLumenGatherCvarState {
    int32 TraceMeshSDFs    = 1;
    float MeshSDFTraceDistance = 180.0f;
    float SurfaceBias      = 5.0f;
    int32 VoxelTracingMode = 0;
    int32 DirectLighting   = 0;
};

extern FLumenGatherCvarState GLumenGatherCvars;
```

### メンバ変数

| 変数名 | 型 | デフォルト | 説明 |
|--------|-----|-----------|------|
| `TraceMeshSDFs` | `int32` | 1 | Mesh SDF トレースを行うか。`r.Lumen.TraceMeshSDFs` の値 |
| `MeshSDFTraceDistance` | `float` | 180.0 | Mesh SDF の最大トレース距離（cm）|
| `SurfaceBias` | `float` | 5.0 | 自己交差防止のサーフェスバイアス（cm）|
| `VoxelTracingMode` | `int32` | 0 | ボクセルトレースモード（0=Global SDF のみ, 1=Voxel を使用）|
| `DirectLighting` | `int32` | 0 | Direct Lighting を個別計算するか（0=Surface Cache に統合, 1=個別計算）|

### 使用箇所

- [[ref_lumen_diffuse_indirect]] — `RenderLumenDiffuseIndirect()` 冒頭で CVar 変更を検出するために参照
- [[ref_lumen_scene]] — CVar 変更時に Lumen シーンの再構築をトリガー

---

## RenderLumenDiffuseIndirect

Lumen Diffuse Indirect の**メインエントリポイント**。

```cpp
void RenderLumenDiffuseIndirect(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    TArray<FViewInfo>& Views,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    const FSceneTextures& SceneTextures,
    FLumenSceneData& LumenSceneData,
    FRDGTextureRef LightingChannelsTexture,
    FAsyncComputeBudget AsyncComputeBudget,
    bool bRenderingToMobileHDR);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（Mesh SDF/TLAS アクセスに使用）|
| `Views` | `TArray<FViewInfo>&` | ビュー配列（通常は 1 要素）|
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | Surface Cache アトラスバッファ |
| `SceneTextures` | `const FSceneTextures&` | GBuffer テクスチャ参照 |
| `LumenSceneData` | `FLumenSceneData&` | Lumen シーンデータ（Mesh SDF 情報等）|
| `LightingChannelsTexture` | `FRDGTextureRef` | ライティングチャンネルマスク（プリミティブ単位）|
| `AsyncComputeBudget` | `FAsyncComputeBudget` | AsyncCompute 使用量の予算 |
| `bRenderingToMobileHDR` | `bool` | Mobile HDR 向けパスか（機能制限あり）|

### 内部処理フロー

1. **有効チェック**
   ```cpp
   if (!LumenDiffuseIndirect::IsAllowed(View, bRenderingToMobileHDR)) { return; }
   // → r.Lumen.DiffuseIndirect.Allow && DoesPlatformSupportLumenGI を確認
   ```

2. **Mesh SDF カリング**
   ```cpp
   // [[ref_lumen_mesh_sdf_culling]] CullForCardTracing() を呼ぶ
   FLumenMeshSDFGridParameters MeshSDFGridParameters;
   CullForCardTracing(GraphBuilder, Scene, View, FrameTemporaries,
       MeshSDFGridParameters, ComputePassFlags);
   ```

3. **トレースパラメータ構築**
   ```cpp
   // [[ref_lumen_tracing_utils]] SetupLumenDiffuseTracingParameters() を呼ぶ
   FLumenIndirectTracingParameters DiffuseTracingParameters;
   SetupLumenDiffuseTracingParameters(View, LumenSceneData, DiffuseTracingParameters);
   ```

4. **AsyncCompute 分岐**
   ```cpp
   ERDGPassFlags ComputePassFlags = UseAsyncCompute(AsyncComputeBudget)
       ? ERDGPassFlags::AsyncCompute
       : ERDGPassFlags::Compute;
   ```

5. **Screen Probe Gather の実行**
   ```cpp
   RenderLumenScreenProbeGather(GraphBuilder, Scene, View, FrameTemporaries,
       SceneTextures, LightingChannelsTexture, MeshSDFGridParameters,
       DiffuseTracingParameters, ComputePassFlags, ...);
   ```

### 使用箇所

- `DeferredShadingRenderer.cpp` — `RenderDiffuseIndirectAndAmbientOcclusion()` から呼ばれる

---

## マイナー関数

> [!note]- LumenDiffuseIndirect::IsAllowed — Lumen GI 実行可否判定
>
> ```cpp
> namespace LumenDiffuseIndirect {
>     bool IsAllowed(const FViewInfo& View, bool bRenderingToMobileHDR);
> }
> ```
>
> **戻り値**: `bool`
>
> **有効条件**:
> - `r.Lumen.DiffuseIndirect.Allow != 0`
> - `DoesPlatformSupportLumenGI(View.GetShaderPlatform())`
> - `bRenderingToMobileHDR == false`（Mobile HDR では無効）
>
> **使用箇所**: [[ref_lumen_diffuse_indirect]] — `RenderLumenDiffuseIndirect()` 冒頭の有効チェック

> [!note]- ShouldRenderLumenDiffuseGI — GI レンダリングが必要か判定
>
> ```cpp
> bool ShouldRenderLumenDiffuseGI(const FSceneViewFamily& ViewFamily);
> ```
>
> **戻り値**: `bool`
>
> `GLumenIrradianceFieldGather != 0` の場合は Irradiance Field 方式で判定、  
> それ以外は Screen Probe Gather の有効フラグを確認。
>
> **使用箇所**: `DeferredShadingRenderer.cpp` — GI パスを追加するか判定するために使用

> [!note]- UseAsyncCompute — AsyncCompute 判定
>
> ```cpp
> bool UseAsyncCompute(FAsyncComputeBudget AsyncComputeBudget);
> ```
>
> `r.Lumen.DiffuseIndirect.AsyncCompute != 0` かつ `AsyncComputeBudget` に余裕がある場合に `true`。
>
> **使用箇所**: [[ref_lumen_diffuse_indirect]] — Compute パスフラグの選択に使用

---

## 主要 CVar

### トレース設定

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.DiffuseIndirect.Allow` | 1 | Lumen GI の有効/無効（0 で完全無効化）|
| `r.Lumen.DiffuseIndirect.TraceStepFactor` | 1.0 | SDF ステップ係数（小さいほど精度増・コスト増）|
| `r.Lumen.DiffuseIndirect.MinSampleRadius` | 10.0 | 最小サンプル半径（cm）|
| `r.Lumen.DiffuseIndirect.MinTraceDistance` | 0.0 | トレース開始距離（cm）|
| `r.Lumen.DiffuseIndirect.SurfaceBias` | 5.0 | 自己交差防止バイアス（cm）|
| `r.Lumen.DiffuseIndirect.CardInterpolateInfluenceRadius` | 10.0 | Surface Cache 補間影響半径 |
| `r.Lumen.DiffuseIndirect.CardTraceEndDistanceFromCamera` | 4000.0 | Surface Cache トレース最大距離（cm）|
| `r.Lumen.TraceDistanceScale` | 1.0 | 全トレース距離のスケール（スケーラビリティ用）|

### Mesh SDF 設定

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.TraceMeshSDFs` | 0 | Mesh SDF トレースの有効/無効（プロジェクト設定で制御）|
| `r.Lumen.TraceMeshSDFs.Allow` | 1 | スケーラビリティ側のMesh SDF 許可フラグ |
| `r.Lumen.TraceMeshSDFs.TraceDistance` | 180.0 | Mesh SDF の最大トレース距離（cm）|

### カリンググリッド設定

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.DiffuseIndirect.CullGridPixelSize` | 64 | フラスタムグリッドセルのピクセルサイズ |
| `r.Lumen.DiffuseIndirect.CullGridDistributionLogZScale` | 0.01 | Z 方向グリッドの対数スケール |
| `r.Lumen.DiffuseIndirect.CullGridDistributionLogZOffset` | 1.0 | Z 方向グリッドのオフセット |
| `r.Lumen.DiffuseIndirect.CullGridDistributionZScale` | 4.0 | Z 方向グリッドの線形スケール |

### 実行設定

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.DiffuseIndirect.AsyncCompute` | 1 | Diffuse GI パスを AsyncCompute で実行するか |

---

## RenderLumenDiffuseIndirect の処理フロー

```
RenderLumenDiffuseIndirect(GraphBuilder, Scene, Views, FrameTemporaries, SceneTextures)
  │
  ├─ LumenDiffuseIndirect::IsAllowed() チェック
  │   → r.Lumen.DiffuseIndirect.Allow && DoesPlatformSupportLumenGI
  │
  ├─ CullForCardTracing()
  │   → Mesh SDF をフラスタムグリッドにカリング
  │   → FLumenMeshSDFGridParameters を構築
  │
  ├─ SetupLumenDiffuseTracingParameters()
  │   → MaxTraceDistance / SurfaceBias / CardTraceEndDistance を設定
  │
  ├─ [AsyncCompute 分岐]
  │   ├─ UseAsyncCompute() = true → Async Compute Queue で実行
  │   └─ false → Graphics Queue で実行
  │
  └─ RenderLumenScreenProbeGather()
        → Screen Probe の配置・トレース・フィルタ・統合
        → 結果を DiffuseIndirect テクスチャに書き込み
```
