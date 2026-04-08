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

```cpp
// リザーバーデータを保持するテクスチャ群（読み取り用 SRV）
struct FReservoirTextures {
    FRDGTexture* RadianceAndWeight;     // xyz=Radiance, w=リザーバーウェイト (W_sum)
    FRDGTexture* HitDistance;           // レイヒット距離
    FRDGTexture* HitNormal;             // ヒット点の法線（8bit 圧縮）
    FRDGTexture* SampleCount;           // 蓄積サンプル数（M 値）
};
```

---

## FReservoirUAVs

```cpp
// リザーバーデータへの書き込み用 UAV
struct FReservoirUAVs {
    FRDGTextureUAV* RadianceAndWeight;
    FRDGTextureUAV* HitDistance;
    FRDGTextureUAV* HitNormal;
    FRDGTextureUAV* SampleCount;
};
```

---

## FReSTIRParameters

```cpp
// ReSTIR パス全体で使用するシェーダーパラメータ
BEGIN_SHADER_PARAMETER_STRUCT(FReSTIRParameters, )
    // リザーバーバッファ（前フレーム・現フレームの2セット）
    SHADER_PARAMETER_STRUCT(FReservoirTextures, PreviousReservoir)  // テンポラル再利用元
    SHADER_PARAMETER_STRUCT(FReservoirTextures, CurrentReservoir)   // 現フレーム入力
    SHADER_PARAMETER_STRUCT(FReservoirUAVs,     OutputReservoir)    // 出力先

    // 再サンプリング判定用閾値
    SHADER_PARAMETER(float, ResamplingNormalDotThreshold)   // 法線類似度閾値（cos角度）
    SHADER_PARAMETER(float, ResamplingDepthErrorThreshold)  // 深度誤差許容率
END_SHADER_PARAMETER_STRUCT()
```

---

## ReSTIR パイプライン

```
LumenReSTIRGather（反射への適用）:

1. [初期サンプリング]
   各ピクセルから 1〜数本の反射レイをトレース
   → リザーバーに (Radiance, ヒット距離, ヒット法線, ウェイト) を格納

2. [テンポラル再サンプリング（Temporal Reuse）]
   PreviousReservoir（前フレーム）と CurrentReservoir をマージ
   ├─ 前フレームのサンプルを現フレームの座標に再投影
   ├─ ResamplingNormalDotThreshold で法線類似度を判定
   ├─ ResamplingDepthErrorThreshold で深度誤差を判定
   └─ 条件を満たす場合のみ前フレームのリザーバーを受け入れ

3. [空間再サンプリング（Spatial Reuse）]
   隣接ピクセルのリザーバーを追加でマージ
   → 各ピクセルが近隣サンプルも活用

4. [シェーディング]
   最終リザーバーの代表サンプルからラジアンスを計算
   → TraceRadiance テクスチャに出力
```

---

## リザーバー更新アルゴリズム

```
リザーバー (W_sum, M, y):
  W_sum: 全サンプルの重み合計
  M: 蓄積サンプル数
  y: 現在の代表サンプル（最大重みのもの）

新サンプル x の受け入れ:
  w_x = target_pdf(x) / source_pdf(x)  ← 重要度ウェイト
  W_sum += w_x
  M += 1
  if rand() < w_x / W_sum:
    y = x  ← 確率的に代表サンプルを更新

最終ウェイト（アンビバイアス補正）:
  W = W_sum / (M * target_pdf(y))
```

---

## テンポラル状態

```cpp
// FLumenViewState 内で管理（LumenViewState.h より）
class FLumenReflectionState {
    // ReSTIR リザーバーの前フレームバッファ
    TRefCountPtr<IPooledRenderTarget> ReservoirRadianceAndWeight;
    TRefCountPtr<IPooledRenderTarget> ReservoirHitDistance;
    TRefCountPtr<IPooledRenderTarget> ReservoirHitNormal;
    TRefCountPtr<IPooledRenderTarget> ReservoirSampleCount;
};
```

---

## 主要 CVar（LumenReSTIRGather.cpp）

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.Reflections.ReSTIR` | 0 | ReSTIR の有効/無効（デフォルトは無効）|
| `r.Lumen.Reflections.ReSTIR.Temporal` | 1 | テンポラル再サンプリングの有効/無効 |
| `r.Lumen.Reflections.ReSTIR.Spatial` | 1 | 空間再サンプリングの有効/無効 |
| `r.Lumen.Reflections.ReSTIR.Spatial.Samples` | 4 | 空間再サンプリングの隣接サンプル数 |
| `r.Lumen.Reflections.ReSTIR.Temporal.MaxSampleCount` | 20 | テンポラル蓄積の最大サンプル数（M クランプ）|
| `r.Lumen.Reflections.ReSTIR.ResamplingNormalDotThreshold` | 0.9 | 法線類似度閾値 |
| `r.Lumen.Reflections.ReSTIR.ResamplingDepthErrorThreshold` | 0.1 | 深度誤差許容率（10%）|
