# GPU: Translucency シェーダー全体概要

- 上位: [[01_gpu_overview]]
- CPU 対応: [[20_translucency_overview]]

---

## 概要

UE5 の Translucency GPU パスは半透明オブジェクトの描画・ライティング・合成を担う。  
フォワードシェーディングパス・ライティングボリューム注入・OIT（Order-Independent Translucency）の3系統で構成される。

---

## GPU シェーダーグループ対応表

| グループ | シェーダーファイル | 主要エントリ | 用途 |
|---------|-----------------|------------|------|
| **a: TranslucentForward** | `BasePassPixelShaders.usf`（Translucent パス）<br>`ComposeSeparateTranslucency.usf` | `BasePassPS`（Translucent）<br>`MainPS`（Compose）| 半透明メッシュのフォワードシェーディング・合成 |
| **b: TranslucentLighting** | `TranslucentLightInjectionShaders.usf`<br>`TranslucentLightingShaders.usf` | `InjectMainPS`<br>`InjectBatchMainCS`<br>`FilterTranslucentVolumeCS` | ライティングボリューム（3D テクスチャ）への光注入・フィルタ |
| **c: OIT** | `OIT/OITSorting.usf`<br>`OITCombine.usf` | `MainCS`（各種）| Order-Independent Translucency（深度ソートなし半透明）|

---

## GPU データフロー

```
【TranslucentForward（Translucent Mesh の描画）】
RenderTranslucency()
  │
  ├─ BasePassPS（Translucent パーミュテーション）
  │   → 深度書き込みなし・アルファブレンド
  │   → SeparateTranslucency テクスチャへ書き込み（半解像度の場合）
  │
  └─ ComposeSeparateTranslucency.usf（MainPS）
      → SceneColor に半透明結果を合成

【TranslucentLighting（ライティングボリューム）】
RenderTranslucencyLighting()
  ├─ InjectMainPS / InjectBatchMainCS（TranslucentLightInjectionShaders.usf）
  │   → ライト輝度を 3D ボリュームテクスチャ（4 段カスケード）に注入
  │
  ├─ FilterTranslucentVolumeCS（TranslucentLightingShaders.usf）
  │   → 隣接ボクセル間でガウスフィルタ
  │
  └─ ボリュームテクスチャ → 半透明マテリアルが Lookup してライティングを適用

【OIT（Order-Independent Translucency）】
RenderOIT()
  ├─ OITSorting.usf（MainCS × 複数パーミュテーション）
  │   → 各ピクセルのサンプルを深度でソート
  │
  └─ OITCombine.usf（MainCS）
      → ソート済みサンプルを前後ブレンドして合成
```

---

## 各グループ詳細

| グループ | 詳細 | リファレンス |
|---------|------|------------|
| a: TranslucentForward | [[a_TranslucentForward/detail_translucent_forward]] | [[a_TranslucentForward/ref_translucent_forward]] |
| b: TranslucentLighting | [[b_TranslucentLighting/detail_translucent_lighting]] | [[b_TranslucentLighting/ref_translucent_lighting]] |
| c: OIT | [[c_OIT/detail_oit]] | [[c_OIT/ref_oit]] |

---

## CPU 側対応

| CPU ドキュメント | 対応 GPU グループ |
|---------------|----------------|
| [[a_translucent_rendering]] — RenderTranslucency・SeparateTranslucency | グループ a |
| [[b_translucent_lighting]] — TranslucentLightVolume・ライト注入 | グループ b |
| [[c_oit]] — OIT パイプライン概要 | グループ c |
