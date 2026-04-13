# リファレンス：SceneRendering.h（FViewInfo / FSceneView）

- グループ: d - View
- 上位: [[23_scene_renderer_overview]]
- 関連: [[ref_scene_renderer]] | [[ref_scene]]
- ソース: `Engine/Source/Runtime/Renderer/Private/SceneRendering.h`
- ソース: `Engine/Source/Runtime/Engine/Public/SceneView.h`

## 概要

`FViewInfo` はレンダースレッド専用のビュークラス。  
エンジン公開 API の `FSceneView` を継承し、  
可視性ビットマップ・DrawList・前フレーム情報等を追加する。

---

## FViewInfo

`SceneRendering.h:1131`

### 基本情報

| 変数 | 型 | 説明 |
|-----|-----|------|
| `ViewRect` | `FIntRect` | スクリーン上のビューピクセル矩形（最終レンダリング領域） |
| `UnscaledViewRect` | `FIntRect` | 動的解像度スケール前の矩形 |
| `ViewState` | `FSceneViewState*` | フレーム間で永続するビュー状態（TAA 履歴等） |
| `Family` | `const FSceneViewFamily*` | 所属ビューファミリー |
| `bIsSceneCapture` | `bool` | SceneCapture2D / Cube のビューか |
| `bIsReflectionCapture` | `bool` | 反射キャプチャか |

### 可視性

| 変数 | 型 | 説明 |
|-----|-----|------|
| `PrimitiveVisibilityMap` | `FSceneBitArray` | 可視プリミティブのビットマップ（`FScene::Primitives` と 1:1） |
| `PrimitiveRayTracingVisibilityMap` | `FSceneBitArray` | レイトレーシング用の可視性マップ |
| `PrimitiveViewRelevanceMap` | `TArray<FPrimitiveViewRelevance, SceneRenderingAllocator>` | パス参加フラグ |
| `NumVisibleDynamicPrimitives` | `int32` | 可視動的プリミティブ数 |

### DrawList

| 変数 | 型 | 説明 |
|-----|-----|------|
| `DynamicMeshElements` | `TArray<FMeshBatchAndRelevance, SceneRenderingAllocator>` | 動的メッシュの DrawList |
| `VisibleMeshDrawCommands` | `FMeshCommandOneFrameArray` | 可視メッシュドローコマンド（StaticMesh 用） |
| `TranslucentPrimCount` | `FTranslucenyPrimCount` | 透過プリミティブのパス別カウンタ |

### ライティング

| 変数 | 型 | 説明 |
|-----|-----|------|
| `VisibleLightInfos` | `TArray<FVisibleLightViewInfo, SceneRenderingAllocator>` | ビュー別の可視ライト情報 |
| `GlobalDistanceFieldInfo` | `FGlobalDistanceFieldInfo` | Lumen / AO 用グローバル SDF |
| `ReflectionCaptureUniformBuffer` | `TUniformBufferRef<FReflectionCaptureShaderData>` | 反射キャプチャ用 UB |

### テンポラル・前フレーム

| 変数 | 型 | 説明 |
|-----|-----|------|
| `PrevViewInfo` | `FPreviousViewInfo` | 前フレームのビュー情報（TAA / モーションブラー用） |
| `ViewMatrices` | `FViewMatrices` | 現在フレームのビュー行列群 |
| `PrevViewMatrices` | `FViewMatrices` | 前フレームのビュー行列群 |
| `TemporalJitterPixels` | `FVector2D` | TAA サブピクセルジッター量 |

### オクルージョン

| 変数 | 型 | 説明 |
|-----|-----|------|
| `HZBOcclusionTester` | `FHZBOcclusionTester` | HZB オクルージョンクエリ管理 |
| `IndividualOcclusionQueries` | `FOcclusionQueryPool` | 個別オクルージョンクエリプール |

### Lumen / レイトレーシング

| 変数 | 型 | 説明 |
|-----|-----|------|
| `RayTracingSceneViewHandle` | `FRayTracingSceneViewHandle` | レイトレーシングシーンのビューハンドル |
| `RayTracingCullingParameters` | `FRayTracingCullingParameters` | レイトレーシング カリングパラメータ |
| `CurrentNaniteVisualizationMode` | `FName` | Nanite 可視化モード名 |

### 主要メソッド

```cpp
// ビュー行列アクセス
const FMatrix& GetProjectionMatrix() const;
const FMatrix& GetViewMatrix() const;
FVector GetViewDirection() const;

// レイトレーシング
void SetRayTracingSceneViewHandle(FRayTracingSceneViewHandle Handle);
FRayTracingSceneViewHandle GetRayTracingSceneViewHandle() const;

// ユーティリティ
FIntPoint GetSecondaryViewRectSize() const;
bool ShouldRenderView() const;
bool IsInstancedStereoPass() const;
```

---

## FPrimitiveViewRelevance

`PrimitiveViewRelevance.h`。プリミティブがどのレンダリングパスに参加するかを示すビットフラグ。

### メンバ変数

```cpp
struct FPrimitiveViewRelevance
{
    uint8 bDrawRelevance        : 1;  // このビューで描画するか
    uint8 bStaticRelevance      : 1;  // StaticMesh パス（DrawList キャッシュ）
    uint8 bDynamicRelevance     : 1;  // DynamicMesh パス（毎フレーム生成）
    uint8 bOpaque               : 1;  // 不透明パス（GBuffer）
    uint8 bMasked               : 1;  // マスク付き不透明パス
    uint8 bTranslucentSurface   : 1;  // 透明サーフェスパス
    uint8 bShadowRelevance      : 1;  // シャドウパス
    uint8 bVelocityRelevance    : 1;  // ベロシティパス
    uint8 bRenderCustomDepth    : 1;  // カスタム深度パス
    uint8 bDecal                : 1;  // デカールパス
    uint8 bHasSimpleLights      : 1;  // シンプルライト（パーティクル等）
    uint8 bEditorPrimitiveRelevance : 1; // エディタプリミティブ
};
```

---

## FSceneViewState

`SceneRendering.h`。`FViewInfo::ViewState` に保持される。フレーム間で永続する。

### 主要メンバ

| 変数 | 型 | 説明 |
|-----|-----|------|
| `TemporalAAHistoryRT` | `TRefCountPtr<IPooledRenderTarget>` | TAA 履歴テクスチャ |
| `PrevSceneColorForTonemapper` | `TRefCountPtr<IPooledRenderTarget>` | 前フレームのシーンカラー |
| `HZB` | `TRefCountPtr<IPooledRenderTarget>` | HZB テクスチャキャッシュ |
| `LumenViewState` | `FLumenViewState` | Lumen のビュー別状態（ラジアンスキャッシュ等） |
| `Scene` | `FScene*` | 所属シーン（nullptr の場合は未リンク） |
| `UniqueID` | `uint32` | ビュー状態の一意 ID |

---

> [!note]- FViewInfo の構築タイミング
> `FViewInfo` は `FSceneRenderer` のコンストラクタで  
> `FSceneView`（ゲームスレッドから渡される）をコピーして生成される:
> ```cpp
> // FSceneRenderer コンストラクタ内
> Views.Emplace(*ViewFamily->Views[ViewIndex]);
> // FSceneView の全メンバをコピーしつつ、
> // PrimitiveVisibilityMap / DynamicMeshElements 等は空の状態で初期化
> ```
> これ以降の `BeginInitViews()` で可視性ビットマップが埋められる。

> [!note]- bIsSceneCapture と ViewState
> `SceneCapture2D` など「フレーム間の継続性が不要なビュー」は  
> `ViewState == nullptr` の状態で生成される。  
> TAA / Lumen 等のテンポラルアルゴリズムは `ViewState != nullptr` の場合のみ動作する。

> [!note]- FViewMatrices の内容
> ```cpp
> struct FViewMatrices
> {
>     FMatrix ViewMatrix;              // ワールド → ビュー
>     FMatrix ProjectionMatrix;        // ビュー → クリップ
>     FMatrix ViewProjectionMatrix;    // ワールド → クリップ
>     FMatrix InvViewMatrix;
>     FMatrix InvProjectionMatrix;
>     FMatrix InvViewProjectionMatrix;
>     FVector4 ScreenPositionScaleBias; // NDC → スクリーン変換
>     FVector PreViewTranslation;       // ワールド精度補正用オフセット
> };
> ```
> `PreViewTranslation` はカメラ原点をワールド原点に移動させる平行移動で、  
> 浮動小数点精度問題（Floating Origin）を軽減するために使用される。
