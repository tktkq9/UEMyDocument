# GPU a Ref: TranslucentForward シェーダーリファレンス

- シェーダー: `BasePassPixelShaders.usf`, `ComposeSeparateTranslucency.usf`
- CPU 対応: [[a_translucent_rendering]]
- 上位: [[01_translucency_gpu_overview]]

---

## シェーダーファイル一覧

| ファイル | エントリポイント | 役割 |
|---------|---------------|------|
| `BasePassPixelShaders.usf` | `BasePassPS`（Translucent パーミュテーション）| 半透明フォワードシェーディング |
| `ComposeSeparateTranslucency.usf` | `MainPS` | SeparateTranslucency → SceneColor 合成 |
| `ComposeSeparateTranslucency.usf` | `CopyBackgroundVisibilityPS` | 背景可視性コピー |
| `TranslucencyUpsampling.usf` | アップサンプリング系 | 半解像度からのアップスケール |

---

## BasePassPS Translucent パーミュテーション

| マクロ | 値 | 意味 |
|--------|-----|------|
| `MATERIALBLENDINGMODE_TRANSLUCENT` | 1 | アルファブレンド |
| `MATERIALBLENDINGMODE_ADDITIVE` | 1 | 加算ブレンド |
| `MATERIALBLENDINGMODE_MODULATE` | 1 | 乗算ブレンド |
| `MATERIAL_SHADINGMODEL_*` | 各種 | シェーディングモデル |
| `TRANSLUCENCY_LIGHTING_MODE` | 0-4 | ライティングモード |

---

## ComposeSeparateTranslucency.usf パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `SceneColorTexture` | `Texture2D` | 入力 SceneColor |
| `LowResDepthTexture` | `Texture2D<float>` | 半解像度深度（アップサンプリング用）|
| `FullResDepthTexture` | `Texture2D<float>` | 全解像度深度 |
| `SeparateTranslucencyPointTexture` | `Texture2D` | SeparateTranslucency（点サンプラー）|
| `SeparateTranslucencyBilinearTexture` | `Texture2D` | SeparateTranslucency（バイリニア）|
| `SeparateModulationPointTexture` | `Texture2D` | Modulate 成分（点サンプラー）|
| `UndistortingDisplacementTexture` | `Texture2D<float2>` | レンズ歪み補正テクスチャ |

---

## パーミュテーション

| マクロ | 値 | 説明 |
|--------|-----|------|
| `PERMUTATION_NEARESTDEPTHNEIGHBOR` | 0/1 | 深度ベース近傍アップサンプリング |

---

## SeparateTranslucency パス制御

| CVar | 説明 |
|------|------|
| `r.SeparateTranslucency` | 0: SceneColor 直接書き込み, 1: 分離バッファ |
| `r.SeparateTranslucencyScreenPercentage` | 分離バッファ解像度（50 = 半解像度）|
| `r.SeparateTranslucencyAutoDownsample` | 自動半解像度切り替え |

---

## ライティングモード（TRANSLUCENCY_LIGHTING_MODE）

| 値 | モード | コスト |
|----|--------|--------|
| 0 | `TLM_VolumetricNonDirectional` | 低（ボリュームサンプル、方向なし）|
| 1 | `TLM_VolumetricDirectional` | 中（ボリューム + 法線）|
| 2 | `TLM_VolumetricPerVertexNonDirectional` | 最低（頂点シェーダーで計算）|
| 3 | `TLM_Surface` | 高（デファードに近い品質）|
| 4 | `TLM_SurfacePerPixelLighting` | 最高（ピクセル単位）|
