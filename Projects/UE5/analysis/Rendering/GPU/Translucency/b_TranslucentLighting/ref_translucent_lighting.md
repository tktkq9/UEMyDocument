# GPU b Ref: TranslucentLighting シェーダーリファレンス

- シェーダー: `TranslucentLightInjectionShaders.usf`, `TranslucentLightingShaders.usf`
- CPU 対応: [[b_translucent_lighting]]
- 上位: [[01_translucency_gpu_overview]]

---

## シェーダーエントリポイント一覧

### TranslucentLightInjectionShaders.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `SimpleLightInjectMainPS` | 91 | PS | SimpleLights の PS 注入 |
| `InjectMainPS` | 166 | PS | 通常ライトのボリューム注入 |
| `InjectBatchMainCS` | 337 | CS | SimpleLights のバッチ CS 注入 |
| `InjectMegaLightsCS` | 455 | CS | MegaLights からの注入 |
| `ClearTranslucentLightingVolumeCS` | 500 | CS | ボリュームのクリア |

### TranslucentLightingShaders.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `WriteToSliceMainVS` | 31 | VS | ボリュームスライス書き込み用 VS |
| `GatherMarkedVoxelsCS` | 102 | CS | 更新ボクセル収集（スパース最適化）|
| `InitIndirectArgsCS` | 129 | CS | Indirect Dispatch 引数初期化 |
| `FilterTranslucentVolumeCS` | 174 | CS | ガウスフィルタ |
| `InjectAmbientCubemapMainPS` | 272 | PS | Ambient Cubemap 注入 |
| `DebugTranslucencyLightingVolumeCS` | 322 | CS | デバッグ可視化 |

---

## InjectMainPS 入出力

| パラメータ | 方向 | 型 | 説明 |
|-----------|------|-----|------|
| `Input.Vertex.UV` | 入力 | `float2` | ボリュームボクセル UV |
| `OutColor0` | 出力 | `float4 : SV_Target0` | SH L0/L1 係数（低周波）|
| `OutColor1` | 出力 | `float4 : SV_Target1` | SH L1 係数（高周波）|

---

## InjectBatchMainCS パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `RWTranslucencyLightingVolumeAmbient` | `RWTexture3D<float4>` | ボリューム UAV（アンビエント）|
| `RWTranslucencyLightingVolumeDirectional` | `RWTexture3D<float4>` | ボリューム UAV（ディレクショナル）|
| `VolumeCascadeIndex` | `uint` | 書き込み先カスケードインデックス |
| `SimpleLightPositionAndRadius` | `float4` | SimpleLight 位置・半径 |
| `SimpleLightColorAndExponent` | `float4` | SimpleLight 色・減衰指数 |

---

## FilterTranslucentVolumeCS パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `TranslucencyLightingVolumeAmbient` | `Texture3D<float4>` | 入力ボリューム（アンビエント）|
| `TranslucencyLightingVolumeDirectional` | `Texture3D<float4>` | 入力ボリューム（ディレクショナル）|
| `RWAmbient` | `RWTexture3D<float4>` | 出力ボリューム（アンビエント）|
| `RWDirectional` | `RWTexture3D<float4>` | 出力ボリューム（ディレクショナル）|

---

## ライティングボリューム仕様

| 項目 | 値 |
|------|----|
| テクスチャ型 | `Texture3D<float4>`（2枚: Ambient + Directional）|
| カスケード数 | 4（TLC_Inner / TLC_Outer × 2 種 × 各スライス）|
| フォーマット | `PF_FloatRGBA`（R16G16B16A16F）|
| 解像度 | `r.TranslucencyLightingVolumeDim`（デフォルト 64 × 64 × 64）|

---

## SH バッファ構造

各ボクセルは **L1 球面調和関数**（4 係数 × RGB）で方向性ライティングを表現する。

| バッファ | 格納内容 |
|--------|---------|
| `OutColor0`（Ambient） | SH L0 係数（スカラー × RGB = 均一ライト）|
| `OutColor1`（Directional） | SH L1 係数（方向 × RGB = 方向性ライト）|

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `TranslucencyVolumeInjectionCommon.ush` | ボリューム座標変換関数・共有型 |
| `DeferredLightingCommon.ush` | ライト評価関数（AttenuationFalloff 等）|
| `VolumeLightingCommon.ush` | ボリュームライティング共通関数 |
| `SkyAtmosphereCommon.ush` | 大気透過率（`USE_CLOUD_TRANSMITTANCE` 時）|

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.TranslucentLightingVolume` | 有効/無効 |
| `r.TranslucencyLightingVolumeDim` | ボリューム解像度 |
| `r.TranslucencyLightingVolumeInnerDistance` | 内側カスケードの距離範囲 |
| `r.TranslucencyLightingVolumeOuterDistance` | 外側カスケードの距離範囲 |
| `r.TranslucencyVolumeBlur` | ガウスフィルタの有効/無効 |
