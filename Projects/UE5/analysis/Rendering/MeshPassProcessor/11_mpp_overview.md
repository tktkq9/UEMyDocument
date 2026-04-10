# MeshPassProcessor 全体概要

- 取得日: 2026-04-10
- 対象: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\`
- 上位: [[01_rendering_overview]]
- Details: [[a_mpp_pipeline]] | [[b_mpp_drawcommand]] | [[c_mpp_shaders]] | [[d_mpp_passes]]
- Reference: [[ref_mpp_processor]] | [[ref_mpp_drawcommand]] | [[ref_mpp_shaders]] | [[ref_mpp_utils]]

---

## MeshPassProcessor とは

UE5 のジオメトリ描画パイプラインのコア。  
マテリアル・シェーダー・レンダーステートを組み合わせて **`FMeshDrawCommand`** を生成し、  
GPU に送るためのパイプラインを担うクラス群。

`FMeshPassProcessor` を継承した各パス固有のクラスが `AddMeshBatch()` を実装し、  
メッシュごとに PSO・バインディングを解決して `FMeshDrawCommand` を生成する。

---

## 全体アーキテクチャ

```mermaid
graph TD
    Scene["FScene\n(全プリミティブ)"] --> StaticMesh["FStaticMeshBatch\n(静的メッシュ)"]
    Scene --> DynMesh["FMeshBatch\n(動的メッシュ)"]

    subgraph MeshPassProcessor["MeshPassProcessor パイプライン"]
        MPP["FMeshPassProcessor\n(派生クラスが AddMeshBatch を実装)"]
        BSM["BuildMeshDrawCommands()\nPSO・バインディング解決"]
        MDC["FMeshDrawCommand\n最終的な描画命令"]
        MPP --> BSM --> MDC
    end

    StaticMesh --> MPP
    DynMesh --> MPP

    MDC --> Cache["静的コマンドキャッシュ\nFCachedMeshDrawCommand"]
    MDC --> Dynamic["動的コマンドリスト\nFMeshCommandOneFrameArray"]

    Cache --> Submit["SubmitMeshDrawCommands\n実際の DrawCall 発行"]
    Dynamic --> Submit
    Submit --> RHI["RHI / GPU"]
```

---

## 静的 vs 動的メッシュコマンド

| 種別 | 対象 | 生成タイミング | キャッシュ |
|------|------|--------------|----------|
| **静的コマンド** | `FStaticMeshBatch` | シーン追加時（1回） | `FCachedMeshDrawCommands` に保持 |
| **動的コマンド** | `FMeshBatch` | 毎フレーム | `FMeshCommandOneFrameArray`（フレーム末に破棄）|

---

## フレームの流れ

```
[シーン追加時]
FPrimitiveSceneInfo::AddStaticMeshes()
  └─ FMeshPassProcessor::AddMeshBatch()
       └─ BuildMeshDrawCommands()
            └─ FMeshDrawCommand 生成 → FCachedMeshDrawCommands に保存

[毎フレーム描画時]
FDeferredShadingRenderer::RenderXxxPass()
  └─ FMeshPassDrawListContext に dynamic MeshBatch を登録
       └─ FMeshPassProcessor::AddMeshBatch()
            └─ BuildMeshDrawCommands()
                 └─ FMeshCommandOneFrameArray に追加

  → SubmitMeshDrawCommands()
       └─ 静的 + 動的コマンドをマージして DrawCall 発行
```

---

## EMeshPass — パス一覧（抜粋）

| パス | 用途 |
|------|------|
| `DepthPass` | デプスプリパス |
| `BasePass` | GBuffer 書き込み（非Nanite） |
| `CSMShadowDepth` | カスケードシャドウマップ |
| `VSMShadowDepth` | Virtual Shadow Maps |
| `Velocity` | ベロシティバッファ |
| `TranslucencyStandard` | 半透明（標準） |
| `CustomDepth` | カスタム深度 |
| `LumenCardCapture` | Lumen Surface Cache 更新 |
| `NaniteMeshPass` | Nanite 通過メッシュ |
| `MeshDecal_DBuffer` | メッシュデカール |
| `HitProxy` (Editor) | エディタ選択判定 |

---

## 主要クラス一覧

| クラス | 役割 |
|--------|------|
| `FMeshPassProcessor` | 全パスの基底クラス。`AddMeshBatch()` をサブクラスが実装 |
| `FMeshDrawCommand` | RHI に直結した描画命令（PSO・バインディング・Draw引数）|
| `FMeshMaterialShader` | メッシュ用マテリアルシェーダーの基底 |
| `FMeshDrawShaderBindings` | シェーダーバインディングのストレージ |
| `FMeshPassProcessorRenderState` | ブレンドステート・深度ステンシルステート |
| `FMeshDrawCommandSortKey` | ドローコールのソートキー（BasePass/Translucent）|

---

## 主要ソースファイル一覧

| ファイル | 役割 |
|---------|------|
| `Public/MeshPassProcessor.h` | 基底クラス・`EMeshPass` enum |
| `Public/MeshPassProcessor.inl` | テンプレートメソッドのインライン実装 |
| `Private/MeshPassProcessor.cpp` | 実装本体 |
| `Private/MeshDrawCommands.h/.cpp` | `FMeshDrawCommand` の生成・実行 |
| `Public/MeshDrawShaderBindings.h` | シェーダーバインディング |
| `Public/MeshMaterialShader.h` | `FMeshMaterialShader` 基底 |
| `Public/MeshPassUtils.h` | `PassProcessorRenderState` ユーティリティ |
| `Public/SimpleMeshDrawCommandPass.h` | `AddSimpleMeshPass()` ヘルパー |
| `Private/MeshDrawCommandStats.h/.cpp` | デバッグ統計 |
