# リファレンス：ScenePrivate.h / Scene.cpp（FScene）

- グループ: c - Scene
- 上位: [[01_rendering_overview]]
- 関連: [[ref_scene_renderer]] | [[ref_view_info]]
- ソース: `Engine/Source/Runtime/Renderer/Private/ScenePrivate.h`
- ソース: `Engine/Source/Runtime/Renderer/Private/Scene.cpp`

## 概要

`FScene` は `UWorld` のレンダースレッド側の表現。  
すべてのレンダリング可能オブジェクト（プリミティブ・ライト等）を保持し、  
`FDeferredShadingSceneRenderer` から `Scene->` でアクセスされる。

---

## FScene

`ScenePrivate.h:~2874`

### プリミティブ関連

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Primitives` | `TArray<FPrimitiveSceneInfo*>` | 登録済みプリミティブ全件 |
| `PrimitiveTransforms` | `TScenePrimitiveArray<FMatrix>` | ワールドトランスフォーム配列 |
| `PrimitiveBounds` | `TScenePrimitiveArray<FPrimitiveBounds>` | AABB + 球バウンド |
| `PrimitiveFlagsCompact` | `TScenePrimitiveArray<FPrimitiveFlagsCompact>` | ビットパックされたプリミティブフラグ |
| `PrimitiveOctree` | `FScenePrimitiveOctree` | 空間分割ツリー（視錐台カリング高速化） |
| `PrimitiveSceneProxies` | `TArray<FPrimitiveSceneProxy*>` | プロキシ配列（Primitives と 1:1 対応） |

### ライト関連

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Lights` | `FLightSceneInfoCompactSparseArray` | 全ライトのスパース配列（コンパクト版） |
| `DirectionalLights` | `TArray<FLightSceneInfo*>` | 平行光（太陽光等）— 最大 `NUM_ATMOSPHERE_LIGHTS` 個 |
| `SkyLight` | `FSkyLightSceneProxy*` | スカイライト（1 シーンに 1 つ） |
| `AtmosphereLights` | `TStaticArray<FLightSceneInfo*, NUM_ATMOSPHERE_LIGHTS>` | 大気散乱用ライト |
| `SimpleDirectionalLight` | `FLightSceneInfo*` | モバイル用シンプル平行光 |

### GPU サブシステム

| 変数 | 型 | 説明 |
|-----|-----|------|
| `GPUScene` | `FGPUScene` | GPU バッファ上の全インスタンスデータ |
| `DefaultLumenSceneData` | `FLumenSceneData*` | Lumen シーン（カード・Surface Cache）|
| `DistanceFieldSceneData` | `FDistanceFieldSceneData` | 全メッシュの SDF データ |
| `RayTracingScene` | `FRayTracingScene` | レイトレーシング BVH（`RHI_RAYTRACING` のみ） |
| `NaniteRasterPipelines` | `FNaniteRasterPipelines[ENaniteMeshPass::Num]` | Nanite ラスタライズパイプライン |
| `NaniteShadingPipelines` | `FNaniteShadingPipelines[ENaniteMeshPass::Num]` | Nanite シェーディングパイプライン |
| `NaniteVisibility` | `FNaniteVisibility[ENaniteMeshPass::Num]` | Nanite 可視性クエリ結果 |
| `SubstrateSceneData` | `FSubstrateSceneData` | Substrate マテリアルシステムデータ |
| `HairStrandsSceneData` | `FHairStrandsSceneData` | ヘアストランドデータ |

### シーン設定

| 変数 | 型 | 説明 |
|-----|-----|------|
| `World` | `UWorld*` | ゲームスレッド側ワールド |
| `FXSystem` | `FFXSystemInterface*` | パーティクル FX システム |
| `EarlyZPassMode` | `EDepthDrawingMode` | Z プリパスの描画モード（シーン全体） |
| `bEarlyZPassMovable` | `bool` | 移動可能オブジェクトを Z プリパスに含めるか |
| `ExponentialFogs` | `TArray<FExponentialHeightFogSceneInfo>` | 指数高さフォグ |
| `Decals` | `TArray<FDeferredDecalProxy*>` | デカール |
| `ViewStates` | `TArray<FSceneViewState*>` | 登録されているビュー状態 |
| `SkyAtmosphereSceneProxy` | `FSkyAtmosphereSceneProxy*` | 大気散乱プロキシ |

### 主要メソッド

```cpp
// プリミティブ管理
void AddPrimitive(UPrimitiveComponent* Primitive);
void RemovePrimitive(UPrimitiveComponent* Primitive);
void UpdatePrimitiveTransform(UPrimitiveComponent* Primitive);
void UpdatePrimitiveLightingAttachment(UPrimitiveComponent* Primitive);

// ライト管理
void AddLight(ULightComponent* Light);
void RemoveLight(ULightComponent* Light);
void UpdateLightTransform(ULightComponent* Light);

// シーン全体
IRendererModule* GetRenderer() const;
ERHIFeatureLevel::Type GetFeatureLevel() const;
EShaderPlatform GetShaderPlatform() const;
FRDGPooledBuffer* GetVirtualShadowMapCacheBuffer() const;
```

---

## FPrimitiveSceneInfo

`PrimitiveSceneInfo.h`。1 プリミティブのレンダリング情報。

### 主要メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Proxy` | `FPrimitiveSceneProxy*` | 描画用データを持つプロキシ |
| `Scene` | `FScene*` | 所属シーン |
| `PrimitiveComponentId` | `FPrimitiveComponentId` | コンポーネント識別子 |
| `PackedIndex` | `int32` | `FScene::Primitives` 内のインデックス |
| `StaticMeshes` | `TArray<FStaticMesh*>` | スタティックメッシュ DrawList |
| `Bounds` | `FBoxSphereBounds` | ワールド空間バウンド |
| `bCacheShadowAsStatic` | `bool` | シャドウをスタティックとしてキャッシュするか |

---

## FGPUScene

`GPUScene.h`。すべてのプリミティブのインスタンスデータを GPU バッファに保持する。

### 主要バッファ

| バッファ | 型 | 説明 |
|---------|-----|------|
| `InstanceSceneDataBuffer` | `FRDGBuffer` | 全インスタンスのトランスフォーム・カスタムデータ |
| `InstancePayloadDataBuffer` | `FRDGBuffer` | インスタンスの追加ペイロードデータ |
| `PrimitiveBuffer` | `FRDGBuffer` | プリミティブ単位のデータ（バウンド等） |

### 主要メソッド

```cpp
// フレームごとのアップロード（Render() 内 :2172 から呼ばれる）
void UploadDynamicPrimitiveShaderDataForView(FRDGBuilder& GraphBuilder, FViewInfo& View);

// プリミティブの追加・削除
void AddPrimitive(FPrimitiveSceneInfo* PrimitiveSceneInfo);
void RemovePrimitive(FPrimitiveSceneInfo* PrimitiveSceneInfo);
```

---

> [!note]- TScenePrimitiveArray — プリミティブデータの SOA レイアウト
> `FScene::PrimitiveTransforms` / `PrimitiveBounds` 等は  
> `TScenePrimitiveArray<T>` というカスタムコンテナを使っている。  
> これは Structure-of-Arrays（SOA）レイアウトで、  
> 視錐台カリングのようなプリミティブ全件走査を SIMD フレンドリーに処理するため。

> [!note]- ENaniteMeshPass — Nanite のマルチパス対応
> ```cpp
> enum class ENaniteMeshPass : uint8
> {
>     BasePass      = 0,   // 通常 GBuffer 描画
>     LumenCardCapture = 1, // Lumen カードキャプチャ用
>     Num
> };
> ```
> `FScene` は `ENaniteMeshPass::Num` 個のパイプラインと可視性クエリを保持する。  
> シャドウ等の追加パスは別途 `FNaniteVisibilityQuery` で管理される。
