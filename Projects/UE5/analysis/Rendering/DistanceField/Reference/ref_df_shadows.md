# Ref: DF Shadows エントリポイントリファレンス

- 対象: `DistanceFieldShadowing.cpp`, `DistanceFieldLightingShared.h`
- 上位: [[10_distance_field_overview]] | [[d_df_shadows]]

---

## エントリポイント

### ShouldPrepareForDistanceFieldShadows

`DistanceFieldShadowing.cpp:764`

```cpp
bool ShouldPrepareForDistanceFieldShadows(const FSceneRenderUpdateInputs& SceneUpdateInputs);
```

以下をすべて満たす場合 `true`：
- `EngineShowFlags.DynamicShadows == true`
- `SupportsDistanceFieldShadows(FeatureLevel, ShaderPlatform)`
- `DoesPlatformSupportDistanceFieldShadowing(ShaderPlatform)`

---

### RayTraceShadows（内部コア）

`DistanceFieldShadowing.cpp:797`

```cpp
void RayTraceShadows(
    FRDGBuilder& GraphBuilder,
    bool bAsyncCompute,
    const FMinimalSceneTextures& SceneTextures,
    ...);
```

Object SDF バッファから Compute Shader でコーントレースを実行する中核関数。  
`DistanceFieldShadowing.usf` シェーダーを使用。  
`FProjectedShadowInfo::RenderRayTracedDistanceFieldProjection` から呼ばれる。

---

### FProjectedShadowInfo::RenderRayTracedDistanceFieldProjection（2 オーバーロード）

`DistanceFieldShadowing.cpp:910, 1121`

```cpp
// オーバーロード 1: RayTracedShadowsTexture を返す（タイル処理などの中間結果）
FScreenPassTexture FProjectedShadowInfo::RenderRayTracedDistanceFieldProjection(
    FRDGBuilder& GraphBuilder,
    bool bAsyncCompute,
    const FMinimalSceneTextures& SceneTextures,
    const FViewInfo& View,
    FIntRect ScissorRect);

// オーバーロード 2: ScreenShadowMaskTexture へ直接合成
void FProjectedShadowInfo::RenderRayTracedDistanceFieldProjection(
    FRDGBuilder& GraphBuilder,
    const FMinimalSceneTextures& SceneTextures,
    FRDGTextureRef ScreenShadowMaskTexture,
    const FViewInfo& View,
    FIntRect ScissorRect,
    FTiledShadowRendering* TiledShadowRendering);
```

---

## 処理フロー

```
RenderRayTracedDistanceFieldProjection()（オーバーロード 2）
  │
  ├─ ScissorRect.Area() > 0 を確認
  ├─ RenderRayTracedDistanceFieldProjection()（オーバーロード 1）を内部呼び出し
  │   → RayTracedShadowsTexture 取得
  │
  └─ RayTracedShadowsTexture → ScreenShadowMaskTexture へ合成
      └─ TiledShadowRendering 有効時は Tiled Indirect Draw を使用
```

---

## FDistanceFieldShadowingParameters（シェーダーパラメータ想定）

`DistanceFieldLightingShared.h` で定義される共有パラメータ構造体（各 SDF Shadow パスが参照）：

| パラメータ | 説明 |
|-----------|------|
| `ObjectParameters` | Object SDF バッファ（位置・スケール・アトラス UV） |
| `SceneObjectBounds` | バウンドボリューム加速構造 |
| `TanLightAngle` | ライトサイズに対応するコーン角 |
| `ShadowRayBias` | セルフシャドウ回避のバイアス |
| `MaxOcclusionDistance` | トレース最大距離 |

---

## Object SDF バッファ構造

```
GPU 上の Object SDF バッファ（FDistanceFieldSceneData::UpdateDistanceFieldObjectBuffers で更新）

Per-Object エントリ:
  - LocalToWorld 行列
  - InvBoxExtent（アトラス UV 変換用）
  - AtlasUVOffset + AtlasUVScale（Mesh SDF Atlas 内の位置）
  - bMostlyTwoSided フラグ
  - Bounds（ワールド空間 AABB）
```

---

## ライト側の設定

ライトコンポーネントで DF Shadows を有効化する：

```cpp
// ULightComponent / UDirectionalLightComponent
UPROPERTY(EditAnywhere, BlueprintReadWrite)
bool bUseRaytracedDistanceFieldShadows; // true: DF Shadows を使用

UPROPERTY(EditAnywhere)
float RaytracedShadowsAmount;           // DF Shadows のブレンド係数
```

---

## 共存ルール

| 組み合わせ | 動作 |
|-----------|------|
| DF Shadows + VSM | ✓ 共存可能（ライト単位で切り替え） |
| DF Shadows + HW Ray Tracing Shadow | ✓ 共存可能（HW RT が優先） |
| DF Shadows + Lumen | ✓ 共存可能（Lumen は独立したシャドウ計算）|
| DF Shadows + `r.DistanceFieldShadowing=0` | ✗ 全ライトで無効化 |

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.DistanceFieldShadowing` | DF Shadows 全体の有効/無効（0: 無効, 1: 有効）|
| `r.DFShadowQuality` | シャドウ品質（ステップ数・コーン精度）|
| `r.DistanceFieldShadowing.UseAsyncCompute` | 非同期 Compute 使用 |
| `r.DistanceFieldShadowing.MaxObjectsPerShadow` | ライトあたりの最大影響オブジェクト数 |
| `r.DistanceFieldShadowing.RayBias` | セルフシャドウ回避バイアス |
