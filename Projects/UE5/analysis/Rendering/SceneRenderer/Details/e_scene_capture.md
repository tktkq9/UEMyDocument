# E: Scene Capture — SceneCaptureComponent → RenderThread レンダリング

- 対象: `SceneCaptureRendering.h/.cpp`, `SceneCaptureComponent.h`, `SceneCaptureComponent2D.h`
- 上位: [[23_scene_renderer_overview]]
- Reference: [[ref_scene_renderer]] | [[ref_view_info]]

---

## 概要

**SceneCapture** はゲームワールドの任意の視点から RenderTarget テクスチャへレンダリングする仕組み。  
`USceneCaptureComponent2D`（2D）/ `USceneCaptureComponentCube`（Cube）の2種があり、  
それぞれ独立した `FSceneView` を持ちレンダラーに投入される。

---

## クラス構成

```
USceneCaptureComponent          （抽象基底 / ShowFlags・CaptureSource・HiddenActors 管理）
├── USceneCaptureComponent2D    （2D テクスチャへのキャプチャ）
└── USceneCaptureComponentCube  （Cube テクスチャへのキャプチャ）
```

---

## `ESceneCaptureSource`（出力フォーマット選択）

```cpp
UENUM()
enum ESceneCaptureSource : int
{
    SCS_SceneColorHDR,          // HDR SceneColor (RGB) + 逆透明度 (A)
    SCS_SceneColorHDRNoAlpha,   // HDR SceneColor (RGB), A=0
    SCS_FinalColorLDR,          // LDR Tonemapped 出力 (RGB)
    SCS_SceneColorSceneDepth,   // HDR SceneColor (RGB) + SceneDepth (A)
    SCS_SceneDepth,             // SceneDepth in R
    SCS_DeviceDepth,            // DeviceDepth in RGB
    SCS_Normal,                 // World Normal (Deferred のみ)
    SCS_BaseColor,              // BaseColor (Deferred のみ)
    SCS_FinalColorHDR,          // HDR Linear Working Color Space
};
```

| CaptureSource | ライティング | 用途例 |
|--------------|------------|--------|
| `SCS_SceneColorHDR` | 有り | ミニマップ・鏡 |
| `SCS_FinalColorLDR` | 有り（Tonemap済み） | UI テクスチャ |
| `SCS_SceneDepth` | 無し | 深度プローブ |
| `SCS_Normal` | 無し（Deferred限定） | マテリアル合成 |

---

## `USceneCaptureComponent` 主要プロパティ

| プロパティ | 型 | 説明 |
|-----------|-----|------|
| `CaptureSource` | `ESceneCaptureSource` | 出力バッファ選択 |
| `bCaptureEveryFrame` | `bool` | true=毎フレーム自動キャプチャ |
| `bCaptureOnMovement` | `bool` | 移動時のみキャプチャ |
| `ShowFlags` / `ShowFlagSettings` | `FEngineShowFlags` | レンダリング機能の On/Off |
| `HiddenActors` | `TArray<AActor*>` | 非表示にする Actor |
| `ShowOnlyActors` | `TArray<AActor*>` | これだけ表示する Actor |
| `MaxViewDistanceOverride` | `float` | LOD 描画距離オーバーライド |

---

## キャプチャのトリガー

```
【自動キャプチャ（毎フレーム）】
bCaptureEveryFrame == true
  → FScene::UpdateAllSceneCaptureContents()  （毎フレーム呼ばれる）
      → USceneCaptureComponent::UpdateSceneCaptureContents(Scene, SceneRenderBuilder)

【手動キャプチャ（Blueprint / C++）】
USceneCaptureComponent2D::CaptureScene()
  → ENQUEUE_RENDER_COMMAND → UpdateSceneCaptureContents(Scene, SceneRenderBuilder)
```

---

## `UpdateSceneCaptureContents` の流れ

```
USceneCaptureComponent2D::UpdateSceneCaptureContents(Scene, SceneRenderBuilder)
  │
  ├─ FSceneView* View を構築
  │   ├─ FSceneViewInitOptions に CaptureSource・ShowFlags・RenderTarget を設定
  │   └─ new FSceneView(InitOptions)
  │
  ├─ FSceneViewFamily を構築
  │   └─ SceneCaptureViewFamily（独自 ViewFamily クラス）
  │
  └─ GetRendererModule().BeginRenderingViewFamily(...)
      └─ FDeferredShadingSceneRenderer::Render() が呼ばれる
         （通常のメインビューと同じパイプラインを独立して実行）
```

---

## ShowFlags によるライティング無効化

```cpp
// SceneCaptureComponent.h
bool IsUnlit() const
{
    return CaptureSource == SCS_SceneDepth
        || CaptureSource == SCS_DeviceDepth
        || CaptureSource == SCS_Normal
        || CaptureSource == SCS_BaseColor
        || (ShowFlags.Lighting == false && ...);
}
```

`IsUnlit()` が true の場合、レンダラーは Lighting パスをスキップし BasePass 出力のみを使用する。

---

## bRenderInMainRenderer（UE5 統合モード）

```cpp
// SceneCaptureComponent2D.h
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=PassInfo)
bool bRenderInMainRenderer;
```

- `false`（デフォルト）: 専用の FSceneView を作りメインレンダラーとは独立して描画
- `true`: メインレンダラーの View に統合（GPU コスト節約。制約あり）

---

## コード実行フロー

```
GameThread
  USceneCaptureComponent2D::CaptureScene()
    ENQUEUE_RENDER_COMMAND
      │
RenderThread
      UpdateSceneCaptureContents(Scene, SceneRenderBuilder)
        │
        ├─ FSceneViewInitOptions 設定（ViewLocation / Rotation / ProjectionMatrix）
        ├─ FSceneView* View = new FSceneView(InitOptions)
        ├─ FSceneViewFamily ViewFamily（独立したフォグ・Exposure 設定）
        └─ IRendererModule::BeginRenderingViewFamily(ViewFamily)
               └─ FDeferredShadingSceneRenderer::Render(RHICmdList)
                      ├─ BeginInitViews()  — キャプチャ対象プリミティブのカリング
                      ├─ RenderBasePass()
                      ├─ RenderLights()    （SCS_SceneColorHDR 時のみ）
                      └─ Tonemap()         （SCS_FinalColorLDR 時のみ）
```

---

## 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|--------|------|
| `USceneCaptureComponent` | `SceneCaptureComponent.h` | 基底：ShowFlags・CaptureSource |
| `USceneCaptureComponent2D` | `SceneCaptureComponent2D.h` | 2D キャプチャ実装 |
| `UpdateSceneCaptureContents(Scene, Builder)` | `SceneCaptureRendering.cpp` | RenderThread 側のビュー投入 |
| `FScene::UpdateAllSceneCaptureContents()` | `Scene.cpp` | 毎フレームの自動キャプチャトリガー |
| `ESceneCaptureSource` | `EngineTypes.h` | 出力バッファ選択 enum |
| `FSceneCaptureViewInfo` | `SceneCaptureComponent.h` | キャプチャ用の最小ビュー情報 |
