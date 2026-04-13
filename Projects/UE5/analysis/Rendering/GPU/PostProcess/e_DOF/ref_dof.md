# リファレンス：DiaphragmDOF.h エントリポイント

- グループ: e - DOF
- 上位: [[detail_dof]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/PostProcess/DiaphragmDOF.h/.cpp`

## 主要関数

### DiaphragmDOF 名前空間

| 関数 | 説明 |
|------|------|
| `IsEnabled(View)` | DiaphragmDOF が有効かどうかを判定 |
| `AddPasses(GraphBuilder, SceneTextures, View, InputSceneColor, EyeAdaptationBuffer)` | DOF 全フェーズ（Setup→Dilate→Gather→Fill→Recombine）を追加 |
| `CircleDofHalfCoc(View)` | View から最大 CoC 半径を取得 |

---

## `FPhysicalCocModel` 全メンバ

| メンバ | 型 | 説明 |
|--------|-----|------|
| `SensorWidth` | `float` | センサー幅（mm） |
| `FStops` | `float` | F 値（絞り径制御） |
| `FocusDistance` | `float` | フォーカス距離（cm） |
| `MaxForegroundCocRadius` | `float` | 手前ボケ最大 CoC ピクセル半径（正値） |
| `MaxBackgroundCocRadius` | `float` | 奥ボケ最大 CoC ピクセル半径（正値） |
| `PetzvalFocusDistance` | `float` | Petzval 収差フォーカス距離 |
| `PetzvalAngle` | `float` | Petzval 旋回ボケ角度 |
| `PetzvalRadialStrength` | `float` | Petzval 放射方向強度 |
| `ComputeCircleDofHalfCoc(Depth)` | `float` | 深度から CoC 半径を計算するメソッド |
| `MinForegroundCocRadius()` | `float` | 手前 CoC 最小値（≤ 0） |

---

## フェーズ別パス関数（内部）

| フェーズ | 関数 | 説明 |
|----------|------|------|
| Setup | `AddDiaphragmDOFSetupPass` | CoC Map + 半解像度 SceneColor 生成 |
| Dilate | `AddDiaphragmDOFCocDilatePass` | Tile 単位で Near/Far CoC を拡張 |
| Gather | `AddDiaphragmDOFGatherPass` | Foreground / Background ボケを Gather |
| Fill | `AddDiaphragmDOFFillHolePass` | Near ボケ境界の穴を補完 |
| Recombine | `AddDiaphragmDOFRecombinePass` | Sharp + Near + Far を合成 |

---

## シェーダーパーミュテーション

```cpp
// DiaphragmDOF.cpp 内の代表的なパーミュテーション次元
class FDOFGatherQualityDim : SHADER_PERMUTATION_ENUM_CLASS("DOF_GATHER_QUALITY", EDOFGatherQuality);
class FDOFLayerProcessingDim : SHADER_PERMUTATION_ENUM_CLASS("DOF_LAYER_PROCESSING", EDOFLayerProcessing);
class FDOFBokehSimulationDim : SHADER_PERMUTATION_ENUM_CLASS("DOF_BOKEH_SIMULATION", EDOFBokehSimulation);
class FDOFRecombineQualityDim : SHADER_PERMUTATION_ENUM_CLASS("DOF_RECOMBINE_QUALITY", EDOFRecombineQuality);
```

| 次元 | 値の例 |
|------|--------|
| `EDOFGatherQuality` | LowQualityAccumulator, HighQualityAccumulator |
| `EDOFLayerProcessing` | ForegroundAndBackground, ForegroundOnly, BackgroundOnly |
| `EDOFBokehSimulation` | Disabled, SimulateLens |
| `EDOFRecombineQuality` | LowQuality, HighQuality |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.DepthOfField.Method` | 1 | 0=BokehDOF, 1=DiaphragmDOF |
| `r.DepthOfField.TemporalAA` | 1 | TAA ジッター連携 |
| `r.DepthOfField.KernelSize` | -1 | 最大カーネル半径（-1=自動） |
| `r.DepthOfField.ScatterOcclusion` | 1 | Scatter 遮蔽 |
| `r.DOF.Gather.AccumulatorQuality` | 1 | Gather 精度 |
| `r.DOF.Gather.PostfilterMethod` | 1 | Gather 後フィルタ |
| `r.DOF.Recombine.Quality` | 2 | Recombine 品質 |
| `r.DOF.TemporalAAQuality` | 1 | TAA 統合品質 |
