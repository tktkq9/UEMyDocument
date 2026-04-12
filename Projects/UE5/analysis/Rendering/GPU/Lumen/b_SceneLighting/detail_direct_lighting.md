# Lumen Scene Direct Lighting シェーダー詳細

- グループ: b - Scene Lighting
- GPU 概要: [[01_lumen_gpu_overview]]
- CPU 詳細: [[b_lumen_scene_lighting]]
- リファレンス: [[ref_direct_lighting]]

---

## 概要

Surface Cache の各 Card に **直接光（DirectLighting）** を書き込むパス群。  
GBuffer に書き込む通常の Deferred Lighting とは独立したパスで、  
Surface Cache アトラス上のテクセルに対してライトの寄与を計算する。

処理は3段階に分かれる：

```
[1] ライトカリング → ライトタイルリストを Card ごとに生成
[2] シャドウマスク → 影の有無をカード単位で計算
[3] ライティング   → LumenCardBatchDirectLightingCS() で Irradiance を計算
    ↓
    DirectLighting Atlas に書き込み
```

---

## パス構成

```
RenderDirectLightingForLumenScene()         // LumenSceneDirectLighting.cpp
  │
  ├─ [1] LumenSceneDirectLightingCulling.usf
  │       → カードに影響するライトをカリング
  │       → LightTilesPerCardTile バッファを生成
  │
  ├─ [2] LumenSceneDirectLightingShadowMask.usf
  │       → 各ライトのシャドウ判定（SW RT または VSM）
  │       → ShadowMaskAtlas を生成
  │
  ├─ [2b] LumenSceneDirectLightingSoftwareRayTracing.usf（SW RT 時）
  │        → SDF を用いてシャドウレイをトレース
  │
  ├─ [2c] LumenSceneDirectLightingHardwareRayTracing.usf（HW RT 時）
  │        → DXR でシャドウレイをトレース
  │
  └─ [3] LumenSceneDirectLighting.usf
          → LumenCardBatchDirectLightingCS()
          → RWDirectLightingAtlas に書き込み
```

---

## LumenSceneDirectLighting.usf — メインライティング CS

### シェーダーコアロジック

```hlsl
[numthreads(CARD_TILE_SIZE, CARD_TILE_SIZE, 1)]
void LumenCardBatchDirectLightingCS(
    uint3 GroupId : SV_GroupID,       // カードタイルインデックス
    uint3 GroupThreadId : SV_GroupThreadID)  // 8×8 テクセルタイル内座標
{
    // 1. CardTile → CardPage → Card → AtlasUV を解決
    // 2. GetSurfaceCacheData() で Albedo / Normal / Depth を取得
    // 3. LightTilesPerCardTile から影響ライト群をループ
    // 4. ReadShadowMaskRay() でシャドウ可否を取得
    // 5. GetIrradianceForLight() で Irradiance を計算
    // 6. RWDirectLightingAtlas[AtlasCoord] += Irradiance
}
```

**CARD_TILE_SIZE** = 8（各スレッドグループは 8×8 = 64 スレッド）

---

## LumenSceneLighting.usf — アトラス合成 + 更新管理

Card 更新の優先度管理とアトラス合成も同じファイルに含まれる。

### CombineLumenSceneLightingCS（最重要）

```hlsl
[numthreads(CARD_TILE_SIZE, CARD_TILE_SIZE, 1)]
void CombineLumenSceneLightingCS(uint3 GroupId, uint3 GroupThreadId)
```

- `AlbedoAtlas × (DirectLightingAtlas + IndirectLightingAtlas)` を計算
- 結果を `RWFinalLightingAtlas` に書き込む
- **FinalLighting = Albedo × (Direct + Indirect) + Emissive**

### 更新優先度管理 CS 群

| エントリポイント | 役割 |
|---------------|------|
| `ClearCardUpdateContextCS()` | 更新コンテキストのリセット |
| `BuildPageUpdatePriorityHistogramCS()` | 各 CardPage の更新優先度ヒストグラムを構築 |
| `SelectMaxUpdateBucketCS()` | 更新するバケットの最大値を決定 |
| `BuildCardsUpdateListCS()` | 実際に更新する Card リストを構築 |
| `SetCardPageIndexIndirectArgsCS()` | Indirect Dispatch の引数を設定 |

### RasterizeToCardsVS

```hlsl
void RasterizeToCardsVS(uint CardIndex : SV_VertexID, out float4 OutPosition : SV_POSITION, out float2 OutCardUV : TEXCOORD0)
```

- Card の UV 範囲を Screen 空間の三角形にマップ
- Lighting パスでカードの描画範囲を正確に制限するために使用

---

## 入出力

### 入力

| バッファ / テクスチャ | 内容 |
|------------------|------|
| `AlbedoAtlas` | Surface Cache の Albedo テクスチャ |
| `NormalAtlas` | Surface Cache の Normal テクスチャ |
| `LightTilesPerCardTile` | カリング済みライトタイルリスト |
| `ShadowMaskAtlas` | シャドウマスク（SW/HW RT で生成）|

### 出力

| テクスチャ | 内容 |
|---------|------|
| `RWDirectLightingAtlas` | 直接光の Irradiance（Surface Cache 上）|
| `RWFinalLightingAtlas` | Direct + Indirect + Emissive の最終合成結果 |

---

## CPU 呼び出しの流れ

```
RenderLumenSceneLighting()              // LumenSceneLighting.cpp:217
  │
  ├─ RenderDirectLightingForLumenScene()  // LumenSceneDirectLighting.cpp
  │   ├─ CullLightsToLumenScene()         → LumenSceneDirectLightingCulling.usf
  │   ├─ RenderDirectLightingShadowMasks() → LumenSceneDirectLightingShadowMask.usf
  │   └─ RenderBatchedLights()            → LumenSceneDirectLighting.usf#LumenCardBatchDirectLightingCS
  │
  ├─ RenderRadiosityForLumenScene()        → Radiosity/LumenRadiosity.usf
  │
  └─ CombineLumenSceneLighting()           → LumenSceneLighting.usf#CombineLumenSceneLightingCS
                                            FinalLightingAtlas に書き込み
```

**CPU 側でのパラメーター割り当て（概略）：**

```cpp
// LumenSceneDirectLighting.cpp 内
FLumenCardBatchDirectLightingCS::FParameters* PassParameters = ...;
PassParameters->RWDirectLightingAtlas = GraphBuilder.CreateUAV(DirectLightingAtlasTexture);
PassParameters->CardTiles = GraphBuilder.CreateSRV(CardTileBuffer);
PassParameters->LightTilesPerCardTile = GraphBuilder.CreateSRV(LightTileBuffer);
// ... シェーダーにバインド後 GraphBuilder.AddPass()
```
