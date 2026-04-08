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
    int32 TraceMeshSDFs    = 1;      // Mesh SDF トレースを行うか
    float MeshSDFTraceDistance = 180.0f; // Mesh SDF の最大トレース距離
    float SurfaceBias      = 5.0f;   // 自己交差防止のサーフェスバイアス
    int32 VoxelTracingMode = 0;      // ボクセルトレースモード
    int32 DirectLighting   = 0;      // Direct Lighting を個別計算するか（0=Surface Cache 統合）
};

extern FLumenGatherCvarState GLumenGatherCvars; // グローバル状態
```

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
