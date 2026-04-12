# REF: Lumen Reflection Denoiser シェーダー

- グループ: f - Reflections
- 詳細: [[detail_reflections]]
- CPU リファレンス: [[ref_lumen_reflections]]
- ソース: `Engine/Shaders/Private/Lumen/LumenReflectionDenoiserClear.usf`  
          `Engine/Shaders/Private/Lumen/LumenReflectionDenoiserTemporal.usf`  
          `Engine/Shaders/Private/Lumen/LumenReflectionDenoiserSpatial.usf`

---

## LumenReflectionDenoiserClear.usf

### 概要

デノイズパス冒頭のクリア処理。

```hlsl
// エントリポイント（詳細はソース参照）
// 反射蓄積バッファ（SpecularAndSecondMoment / NumFramesAccumulated）を初期化
```

---

## LumenReflectionDenoiserTemporal.usf

### LumenReflectionDenoiserTemporalCS

```hlsl
[numthreads(REFLECTION_THREADGROUP_SIZE_2D, REFLECTION_THREADGROUP_SIZE_2D, 1)]
void LumenReflectionDenoiserTemporalCS(uint3 GroupId, uint3 GroupThreadId)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 前フレームの反射履歴を現フレームとモーションベクターでリプロジェクトしてブレンド |
| **入力** | `SpecularAndSecondMoment`（現フレーム）, `HistorySpecularAndSecondMoment`（前フレーム）, `SceneVelocity`, `SceneDepth` |
| **出力** | `RWSpecularAndSecondMoment`（float4: RGB + 2nd モーメント）, `RWNumFramesAccumulated`（UNORM float）|
| **CPU 関数** | `DenoiseReflections_Temporal()` |

#### テンポラルブレンドロジック

```hlsl
// 2次モーメントで輝度分散を計算 → 分散が大きいほど HistoryWeight を下げる
float Variance = SecondMoment - Square(Luminance);
float HistoryWeight = lerp(MaxHistoryWeight, MinHistoryWeight, saturate(Variance * VarianceBlendFactor));
// 高輝度変化（ライト移動等）で Ghost アーティファクトを抑制
```

---

## LumenReflectionDenoiserSpatial.usf

### LumenReflectionDenoiserSpatialCS

```hlsl
[numthreads(REFLECTION_THREADGROUP_SIZE_2D, REFLECTION_THREADGROUP_SIZE_2D, 1)]
void LumenReflectionDenoiserSpatialCS(uint3 GroupId, uint3 GroupThreadId)
```

| 項目 | 内容 |
|-----|------|
| **目的** | テンポラル蓄積後の反射テクスチャをバイラテラルフィルタで空間的にデノイズ |
| **入力** | `SpecularAndSecondMoment`（テンポラル済み）, `SceneDepth`, `SceneNormal`, `GBufferB`（Roughness）|
| **出力** | `RWSpecularIndirectAccumulated`（float3）— DiffuseIndirectComposite が参照する最終反射 |
| **CPU 関数** | `DenoiseReflections_Spatial()` |

#### フィルタリング重み計算

```hlsl
// バイラテラル重み：以下すべての条件で隣接ピクセルの重みを決定
float DepthWeight   = exp(-abs(CenterDepth - NeighborDepth) * DepthScale);
float NormalWeight  = pow(saturate(dot(CenterNormal, NeighborNormal)), NormalPower);
float RoughnessWeight = exp(-abs(CenterRoughness - NeighborRoughness) * RoughnessScale);
float TotalWeight = DepthWeight * NormalWeight * RoughnessWeight;
// TotalWeight が低い（境界部・異素材）ほど隣接ピクセルを参照しない
```

#### 透過オブジェクトへの追記

```hlsl
// Substrate の不透明粗面屈折（OpaqueRoughRefraction）が有効な場合
// RWTranslucencyLighting にも反射を書き込む
RWTranslucencyLighting[PixelPos] += ReflectionContribution;
```

---

## CPU 呼び出しの流れ

```
RenderLumenReflections()
  │
  ├─ ClearReflectionDenoiserHistory()       → LumenReflectionDenoiserClear.usf
  ├─ ...（トレース・Resolve）
  ├─ LumenReflectionDenoiserTemporalCS      ← テンポラル蓄積
  └─ LumenReflectionDenoiserSpatialCS       ← スペーシャルフィルタ
       ↓
  RWSpecularIndirectAccumulated
    → DiffuseIndirectComposite の DiffuseIndirect_Lumen_3 として渡される
```

---

> [!note]- 2次モーメントによる適応的テンポラルブレンド
> テンポラルデノイザは単純に「現フレーム × α + 履歴 × (1-α)」でブレンドするのではなく、輝度の **2次モーメント**（分散の推定量）を毎フレーム蓄積して Ghost（前フレームの残像）を動的に検出する。  
> 分散が大きい（急激な輝度変化）ピクセルではブレンドアルファを上げて現フレームを優先し、分散が小さい（安定）ピクセルでは履歴を多く使うことでノイズを抑える。

> [!note]- スペーシャルフィルタとバイラテラル重みの限界
> バイラテラルフィルタは Depth・Normal・Roughness が近い隣接ピクセルのみを統合するため、エッジや素材境界での漏れ（Light Bleeding）を防げる。  
> ただし Roughness の高い（拡散反射に近い）サーフェスでは近傍ピクセルの反射方向が大きく異なるため、広いカーネルでのフィルタリングが難しく、テンポラル蓄積への依存度が上がる。
