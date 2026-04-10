# リファレンス：LumenReSTIRGather.h / LumenReSTIRGather.cpp

- グループ: f - Reflections
- 上位: [[f_lumen_reflections]]
- 関連: [[ref_lumen_reflections]] | [[ref_lumen_reflection_tracing]] | [[ref_lumen_reflection_hwrt]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenReSTIRGather.h`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenReSTIRGather.cpp`

---

## 概要

Lumen Reflections における **ReSTIR（Reservoir-based Spatio-Temporal Importance Resampling）** 実装ファイル。  
反射レイのサンプル再利用で品質を向上させる。  
リザーバー（重みつきサンプル履歴）を時間・空間両方向で再サンプリングし、  
少ないレイ本数で高品質な反射を実現する。

---

## FReservoirTextures

リザーバーデータを保持するテクスチャ群（読み取り用 SRV）。

```cpp
struct FReservoirTextures {
    FRDGTexture* RadianceAndWeight;
    FRDGTexture* HitDistance;
    FRDGTexture* HitNormal;
    FRDGTexture* SampleCount;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `RadianceAndWeight` | `FRDGTexture*` | `float4` テクスチャ。xyz=Radiance（反射色）, w=リザーバーウェイト W_sum |
| `HitDistance` | `FRDGTexture*` | 反射レイのヒット距離（テンポラル再投影の信頼性判定に使用）|
| `HitNormal` | `FRDGTexture*` | ヒット点の法線（8bit 圧縮。`ResamplingNormalDotThreshold` の比較対象）|
| `SampleCount` | `FRDGTexture*` | 蓄積サンプル数 M（テンポラル蓄積量の上限クランプに使用）|

### 使用箇所

- [[ref_lumen_restir]] — `FReSTIRParameters::PreviousReservoir` / `CurrentReservoir` として格納
- [[ref_lumen_restir]] — `FLumenReflectionState` に前フレームバッファとして永続保持

---

## FReservoirUAVs

リザーバーデータへの書き込み用 UAV。

```cpp
struct FReservoirUAVs {
    FRDGTextureUAV* RadianceAndWeight;
    FRDGTextureUAV* HitDistance;
    FRDGTextureUAV* HitNormal;
    FRDGTextureUAV* SampleCount;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `RadianceAndWeight` | `FRDGTextureUAV*` | `RadianceAndWeight` テクスチャへの書き込み UAV |
| `HitDistance` | `FRDGTextureUAV*` | ヒット距離テクスチャへの書き込み UAV |
| `HitNormal` | `FRDGTextureUAV*` | ヒット法線テクスチャへの書き込み UAV |
| `SampleCount` | `FRDGTextureUAV*` | サンプル数テクスチャへの書き込み UAV |

### 使用箇所

- [[ref_lumen_restir]] — `FReSTIRParameters::OutputReservoir` として格納
- [[ref_lumen_restir]] — テンポラル / 空間再サンプリングパスが結果を書き込む

---

## FReSTIRParameters

ReSTIR パス全体で使用するシェーダーパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FReSTIRParameters, )
    SHADER_PARAMETER_STRUCT(FReservoirTextures, PreviousReservoir)
    SHADER_PARAMETER_STRUCT(FReservoirTextures, CurrentReservoir)
    SHADER_PARAMETER_STRUCT(FReservoirUAVs,     OutputReservoir)
    SHADER_PARAMETER(float, ResamplingNormalDotThreshold)
    SHADER_PARAMETER(float, ResamplingDepthErrorThreshold)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `PreviousReservoir` | `FReservoirTextures` | 前フレームのリザーバー（テンポラル再利用元）|
| `CurrentReservoir` | `FReservoirTextures` | 現フレームの初期サンプルリザーバー（入力）|
| `OutputReservoir` | `FReservoirUAVs` | マージ後のリザーバー（出力先）|
| `ResamplingNormalDotThreshold` | `float` | 法線類似度のしきい値（cos角度）。デフォルト: 0.9（約26度以内のみ受け入れ）|
| `ResamplingDepthErrorThreshold` | `float` | 深度誤差の許容率。デフォルト: 0.1（10%以内のみ受け入れ）|

### 使用箇所

- [[ref_lumen_restir]] — テンポラル再サンプリング CS と空間再サンプリング CS に渡される

---

## FLumenReflectionState（テンポラル状態）

```cpp
// FLumenViewState 内で管理（LumenViewState.h より）
class FLumenReflectionState {
    TRefCountPtr<IPooledRenderTarget> ReservoirRadianceAndWeight;
    TRefCountPtr<IPooledRenderTarget> ReservoirHitDistance;
    TRefCountPtr<IPooledRenderTarget> ReservoirHitNormal;
    TRefCountPtr<IPooledRenderTarget> ReservoirSampleCount;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `ReservoirRadianceAndWeight` | `IPooledRenderTarget` | 前フレームの RadianceAndWeight テクスチャ（フレーム間で永続）|
| `ReservoirHitDistance` | `IPooledRenderTarget` | 前フレームのヒット距離テクスチャ |
| `ReservoirHitNormal` | `IPooledRenderTarget` | 前フレームのヒット法線テクスチャ |
| `ReservoirSampleCount` | `IPooledRenderTarget` | 前フレームの蓄積サンプル数テクスチャ |

### 使用箇所

- `FLumenViewState` — `LumenViewState.h` にメンバとして定義
- [[ref_lumen_restir]] — `FReSTIRParameters::PreviousReservoir` の構築時に外部テクスチャとして登録

---

## ReSTIR パイプライン

```
LumenReSTIRGather（反射への適用）:

1. [初期サンプリング]
   各ピクセルから 1〜数本の反射レイをトレース（SW RT / HW RT）
   → リザーバーに (Radiance, ヒット距離, ヒット法線, ウェイト) を格納
   → CurrentReservoir に書き込み

2. [テンポラル再サンプリング（Temporal Reuse）]
   r.Lumen.Reflections.ReSTIR.Temporal = 1 の場合:
   PreviousReservoir（前フレーム）と CurrentReservoir をマージ
   ├─ 前フレームのサンプルを現フレームの座標に再投影（モーションベクターを使用）
   ├─ ResamplingNormalDotThreshold で法線類似度を判定
   ├─ ResamplingDepthErrorThreshold で深度誤差を判定
   └─ 条件を満たす場合のみ前フレームのリザーバーを受け入れ
      M クランプ: MaxSampleCount = 20（r.Lumen.Reflections.ReSTIR.Temporal.MaxSampleCount）

3. [空間再サンプリング（Spatial Reuse）]
   r.Lumen.Reflections.ReSTIR.Spatial = 1 の場合:
   隣接ピクセル（Samples = 4）のリザーバーを追加でマージ
   → 各ピクセルが近隣サンプルも活用して品質向上
   → ResamplingNormal / Depth しきい値で不適切な隣接を除外

4. [シェーディング]
   最終リザーバーの代表サンプル y からラジアンスを計算
   → TraceRadiance テクスチャに出力
   → 後続の Resolve / テンポラル蓄積に渡す
```

---

## リザーバー更新アルゴリズム

```
リザーバー (W_sum, M, y):
  W_sum: 全サンプルの重み合計
  M: 蓄積サンプル数（MaxSampleCount でクランプ）
  y: 現在の代表サンプル（最大重みのもの）

新サンプル x の受け入れ:
  w_x = target_pdf(x) / source_pdf(x)  ← 重要度ウェイト
  W_sum += w_x
  M += 1
  if rand() < w_x / W_sum:
    y = x  ← 確率的に代表サンプルを更新

最終ウェイト（アンビバイアス補正）:
  W = W_sum / (M * target_pdf(y))
  → ラジアンス計算: Radiance_final = y.Radiance * W
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.Reflections.ReSTIR` | 0 | ReSTIR の有効/無効（デフォルトは無効）|
| `r.Lumen.Reflections.ReSTIR.Temporal` | 1 | テンポラル再サンプリングの有効/無効 |
| `r.Lumen.Reflections.ReSTIR.Spatial` | 1 | 空間再サンプリングの有効/無効 |
| `r.Lumen.Reflections.ReSTIR.Spatial.Samples` | 4 | 空間再サンプリングの隣接サンプル数 |
| `r.Lumen.Reflections.ReSTIR.Temporal.MaxSampleCount` | 20 | テンポラル蓄積の最大サンプル数（M クランプ）|
| `r.Lumen.Reflections.ReSTIR.ResamplingNormalDotThreshold` | 0.9 | 法線類似度閾値（cos θ）|
| `r.Lumen.Reflections.ReSTIR.ResamplingDepthErrorThreshold` | 0.1 | 深度誤差許容率（10%）|
