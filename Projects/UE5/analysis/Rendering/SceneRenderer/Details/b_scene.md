# B: FScene — シーンデータ管理

- 対象: `ScenePrivate.h`, `Scene.cpp`
- 上位: [[23_scene_renderer_overview]]
- Reference: [[ref_scene]]

---

## クラス概要

`FScene` はゲームスレッド側の `UWorld` に対応するレンダースレッド側のシーンデータ。  
すべてのプリミティブ・ライト・サブシステムデータを保持し、  
`FDeferredShadingSceneRenderer` から直接参照される。

```
UWorld（ゲームスレッド）
  └─ FScene（レンダースレッド）  ← ScenePrivate.h:2874
      ├─ FPrimitiveSceneInfo[]    — 各メッシュの描画情報
      ├─ FLightSceneInfo[]        — ライト情報
      ├─ FGPUScene                — GPU バッファ上のインスタンスデータ
      ├─ FLumenSceneData*         — Lumen シーン
      ├─ FDistanceFieldSceneData  — SDF データ
      └─ FRayTracingScene         — レイトレーシング BVH
```

---

## 主要メンバ変数

### プリミティブ関連

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Primitives` | `TArray<FPrimitiveSceneInfo*>` | 登録済みプリミティブ（メッシュ・パーティクル等） |
| `PrimitiveTransforms` | `TScenePrimitiveArray<FMatrix>` | 各プリミティブのワールドトランスフォーム |
| `PrimitiveBounds` | `TScenePrimitiveArray<FPrimitiveBounds>` | AABB + 球バウンド |
| `PrimitiveOctree` | `FScenePrimitiveOctree` | 空間分割（視錐台カリング高速化） |

### ライト関連

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Lights` | `FLightSceneInfoCompactSparseArray` | 全ライトのスパース配列 |
| `DirectionalLights` | `TArray<FLightSceneInfo*>` | 平行光（太陽光等） |
| `SkyLight` | `FSkyLightSceneProxy*` | スカイライト |
| `AtmosphereLights` | `TStaticArray<FLightSceneInfo*, NUM_ATMOSPHERE_LIGHTS>` | 大気散乱用ライト |

### サブシステム

| 変数 | 型 | 説明 |
|-----|-----|------|
| `GPUScene` | `FGPUScene` | GPU バッファ上のインスタンスデータ全体 |
| `DefaultLumenSceneData` | `FLumenSceneData*` | Lumen シーン（カード・Surface Cache） |
| `DistanceFieldSceneData` | `FDistanceFieldSceneData` | メッシュ SDF データ |
| `RayTracingScene` | `FRayTracingScene` | レイトレーシング BVH（RHI_RAYTRACING のみ） |
| `SubstrateSceneData` | `FSubstrateSceneData` | Substrate マテリアルシステムデータ |

### その他

| 変数 | 型 | 説明 |
|-----|-----|------|
| `World` | `UWorld*` | 対応するゲームスレッドワールド |
| `FXSystem` | `FFXSystemInterface*` | パーティクル FX システム |
| `EarlyZPassMode` | `EDepthDrawingMode` | Z プリパスの描画モード |
| `Decals` | `TArray<FDeferredDecalProxy*>` | デカール配列 |
| `ExponentialFogs` | `TArray<FExponentialHeightFogSceneInfo>` | 指数高さフォグ |
| `NaniteRasterPipelines` | `FNaniteRasterPipelines[ENaniteMeshPass::Num]` | Nanite ラスタライズパイプライン |
| `NaniteShadingPipelines` | `FNaniteShadingPipelines[ENaniteMeshPass::Num]` | Nanite シェーディングパイプライン |

---

## FPrimitiveSceneInfo

シーン内の 1 プリミティブ（メッシュコンポーネント等）のレンダースレッド表現。

```cpp
class FPrimitiveSceneInfo
{
public:
    FPrimitiveSceneProxy* Proxy;          // 描画用データ（スタティックメッシュなら FStaticMeshSceneProxy）
    FScene* Scene;
    FPrimitiveComponentId PrimitiveComponentId;

    TArray<FStaticMesh*> StaticMeshes;    // キャッシュ済みスタティックメッシュ
    int32 PackedIndex;                    // FScene::Primitives 内のインデックス
};
```

---

## FGPUScene — GPU ドリブンインスタンスデータ

すべてのプリミティブのインスタンスデータ（トランスフォーム・カスタムデータ等）を  
GPU 側のバッファ（`GPUScene.h`）に保持する。  
Nanite / Lumen / VSM が共通して参照する。

```cpp
// フレームごとにダーティなプリミティブのデータをアップロード
Scene->GPUScene.UploadDynamicPrimitiveShaderDataForView(GraphBuilder, View);
```

---

## コード実行フロー

### エントリポイント

```
──── シーンへの追加（ゲームスレッド） ────

UPrimitiveComponent::CreateRenderState_Concurrent()
  └─ GetWorld()->Scene->AddPrimitive(this)
      └─ ENQUEUE_RENDER_COMMAND → FScene::AddPrimitive_RenderThread()
          ├─ FPrimitiveSceneInfo* を生成
          ├─ FScene::Primitives に追加
          ├─ PrimitiveOctree に挿入
          └─ GPUScene.AddPrimitive() → GPU バッファにアップロード予約

──── フレームごとの GPU アップロード（レンダースレッド） ────

FDeferredShadingSceneRenderer::Render()
  └─ Scene->GPUScene.UploadDynamicPrimitiveShaderDataForView()   :2172
      ├─ ダーティプリミティブのデータを GPU バッファに転送
      └─ FRDGBuffer（InstanceSceneData）に書き込む

──── Lumen シーン更新 ────

FDeferredShadingSceneRenderer::Render()
  └─ BeginUpdateLumenSceneTasks()   :1873
      ├─ 変更されたカードのリキャプチャスケジュール
      └─ UpdateLumenScene() → FScene::DefaultLumenSceneData を更新
```

### フロー詳細

1. **プリミティブのライフタイム**
   ```
   ゲームスレッド:  AddPrimitive() → RemovePrimitive()
   レンダースレッド: FPrimitiveSceneInfo の生成 / 削除が同期される
   ```
   - `FPrimitiveSceneProxy` はゲームスレッドのコンポーネントから  
     `CreateSceneProxy()` で生成され、レンダースレッドに渡される

2. **GPUScene の更新タイミング**
   - 追加・削除・トランスフォーム変更の度にダーティフラグが立つ
   - 毎フレーム `Render()` 内の `:2172` でバッチアップロードされる
   - Nanite / Lumen は `GPUScene.InstanceSceneData` を GPU シェーダーで直接参照する

3. **FScene と FSceneRenderer の関係**
   ```cpp
   // FSceneRenderer が FScene を持つ（1フレームに1つ生成）
   class FSceneRenderer
   {
       FScene* Scene;  // ← 同じシーンをフレームをまたいで参照
   };
   ```

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|--------|------|
| `FScene` | `ScenePrivate.h` | [[ref_scene]] レンダースレッド側シーンデータ全体 |
| `FPrimitiveSceneInfo` | `PrimitiveSceneInfo.h` | 個々のプリミティブのレンダリング情報 |
| `FPrimitiveSceneProxy` | `PrimitiveSceneProxy.h` | プリミティブの描画プロキシ |
| `FGPUScene` | `GPUScene.h` | GPU バッファ上のインスタンスデータ |
| `FLumenSceneData` | `LumenSceneData.h` | Lumen シーン（カード・Surface Cache） |
| `FDistanceFieldSceneData` | `DistanceFieldAmbientOcclusion.h` | SDF データ管理 |
| `FScene::AddPrimitive()` | `Scene.cpp` | プリミティブ登録エントリ |
| `FGPUScene::UploadDynamicPrimitiveShaderDataForView()` | `GPUScene.cpp` | フレームごとの GPU アップロード |
