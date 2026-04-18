# GPU b: TranslucentLighting — ライティングボリューム注入・フィルタ

- シェーダー: `TranslucentLightInjectionShaders.usf`, `TranslucentLightingShaders.usf`
- CPU 対応: [[b_translucent_lighting]]
- 上位: [[01_translucency_gpu_overview]]

---

## 概要

半透明オブジェクトのライティングはデファードのライティングバッファを直接参照できない。  
代わりに「ライティングボリューム（3D テクスチャカスケード）」へ各ライトの輝度を注入し、  
マテリアルシェーダーがこのボリュームをサンプリングすることでライティングを受け取る。

---

## ライティングボリューム構成

```
TranslucencyLightingVolume（3D テクスチャ × 4 カスケード）
  ├─ Cascade 0: 近距離（高解像度）
  ├─ Cascade 1: 中距離
  ├─ Cascade 2: 遠距離
  └─ Cascade 3: 最遠（低解像度）

各カスケードはカメラ中心の AABB ボリューム（r.TranslucencyLightingVolumeDim で解像度制御）
```

---

## InjectMainPS（TranslucentLightInjectionShaders.usf: line 166）

ポイントライト・スポットライトの輝度をボリュームへ注入する PS。  
ライト種別ごとにインスタンシングで各スライスへ書き込む。

```hlsl
void InjectMainPS(
    FWriteToSliceGeometryOutput Input,
    out float4 OutColor0 : SV_Target0,  // Directional（低周波 SH）
    out float4 OutColor1 : SV_Target1)  // Directional（高周波 SH）
{
    // ピクセルのワールド位置を計算（ボリューム座標 → ワールド座標）
    // ライトの輝度・減衰・シャドウを計算（DeferredLightingCommon.ush）
    // SH（球面調和関数）係数に変換して2バッファへ出力
}
```

---

## InjectBatchMainCS（TranslucentLightInjectionShaders.usf: line 337）

SimpleLights（粒子ライト等の軽量ライト）を Compute Shader でバッチ注入する。

```hlsl
[numthreads(INJECT_BATCH_THREADGROUP_SIZE, 1, 1)]
void InjectBatchMainCS(uint3 ThreadId : SV_DispatchThreadID)
{
    // 複数の SimpleLights をまとめてボリュームに注入
    // 各スレッドが1ボクセルを担当
}
```

---

## InjectMegaLightsCS（TranslucentLightInjectionShaders.usf: line 455）

MegaLights システムからの輝度を半透明ライティングボリュームへ注入する。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void InjectMegaLightsCS(uint3 ThreadId : SV_DispatchThreadID)
{
    // MegaLights の評価済み輝度をボリュームボクセルに注入
}
```

---

## FilterTranslucentVolumeCS（TranslucentLightingShaders.usf: line 174）

ライティングボリュームにガウスフィルタをかけて滑らかにする。

```hlsl
[numthreads(FILTER_THREADGROUP_SIZE, FILTER_THREADGROUP_SIZE, 1)]
void FilterTranslucentVolumeCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    // 隣接ボクセルをガウス重みで加重平均
    // 各カスケードに対して X, Y, Z 軸それぞれで1パス
}
```

---

## GatherMarkedVoxelsCS（TranslucentLightingShaders.usf: line 102）

更新が必要なボクセルを収集するCS（スパース更新最適化）。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void GatherMarkedVoxelsCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    // マーク済みボクセルのみを IndirectArgs リストに収集
    // → FilterTranslucentVolumeCS で Indirect Dispatch に使用
}
```

---

## ClearTranslucentLightingVolumeCS（line 500）

毎フレームの注入前にボリュームをゼロクリアする。

---

## WriteToSliceMainVS（TranslucentLightingShaders.usf: line 31）

ボリュームの各スライスに描画するためのインスタンス化 VS。  
`SV_InstanceID` を使ってレンダーターゲット配列の Z スライスを選択する。

```hlsl
void WriteToSliceMainVS(
    float2 InPosition : ATTRIBUTE0,
    float2 InUV       : ATTRIBUTE1,
    uint LayerIndex : SV_InstanceID,
    out FWriteToSliceVertexOutput Output)
{
    Output.LayerIndex = LayerIndex + MinZ; // SV_RenderTargetArrayIndex 用
}
```

---

## InjectAmbientCubemapMainPS（line 272）

Ambient Cubemap の輝度をボリュームに注入する。  
Sky Light が無効な場合のフォールバックとして機能する。

---

## マテリアルシェーダーとの接続

```hlsl
// BasePassPixelShaders.usf（半透明 + VolumeLighting モード時）
float3 GetTranslucencyLighting(FMaterialPixelParameters Params)
{
    // ワールド座標 → ボリューム UV
    // 対応カスケードのボリュームテクスチャをサンプリング
    // SH → 法線方向で Irradiance 計算
    return SampleTranslucencyLightingVolume(WorldPos, WorldNormal);
}
```

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.TranslucencyLightingVolumeDim` | ライティングボリューム解像度（デフォルト 64）|
| `r.TranslucencyLightingVolumeInnerDistance` | 内側カスケードのカバー距離 |
| `r.TranslucencyLightingVolumeOuterDistance` | 外側カスケードのカバー距離 |
| `r.TranslucentLightingVolume` | ライティングボリュームの有効/無効 |
