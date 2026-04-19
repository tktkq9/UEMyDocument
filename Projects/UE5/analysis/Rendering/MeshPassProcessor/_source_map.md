# MeshPassProcessor ソースマップ

- 対象: UE5 メッシュ描画パイプラインのコア（`FMeshPassProcessor` 中心）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[11_mpp_overview]]

`FMeshPassProcessor` は EMeshPass ごとにサブクラスが存在し、
`FMeshDrawCommand` を生成して静的キャッシュまたは動的フレームリストへ積む。

---

## ソースパス

| 対象 | パス |
|------|------|
| 公開ヘッダ | `Engine/Source/Runtime/Renderer/Public/MeshPassProcessor*.h` |
| 実装 | `Engine/Source/Runtime/Renderer/Private/MeshPassProcessor.cpp`, `MeshDrawCommands.*` |
| 静的キャッシュ呼び出し元 | `Engine/Source/Runtime/Renderer/Private/PrimitiveSceneInfo.cpp` |

---

## ファイル → クラス対応

### 基底・Processor

| ファイル | 主要クラス / enum | 役割 | 参照 |
|---------|----------------|------|------|
| `Public/MeshPassProcessor.h` | `FMeshPassProcessor`, `EMeshPass`, `FPassProcessorManager` | 基底・EMeshPass 定義・プロセッサファクトリ | [[Reference/ref_mpp_processor]] |
| `Public/MeshPassProcessor.inl` | テンプレート `BuildMeshDrawCommands<>` | インライン実装 | 同 |
| `Private/MeshPassProcessor.cpp` | `SubmitMeshDrawCommands`:1604, `SubmitMeshDrawCommandsRange`:1616, `FinalizeCommand`:2032 | 実装本体・DrawCall 発行 | [[Details/a_mpp_pipeline]] |

### DrawCommand

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `Private/MeshDrawCommands.h` | `FMeshDrawCommand`, `FMeshCommandOneFrameArray`, `FVisibleMeshDrawCommandProcessTask` | コマンド構造・一時配列 | [[Reference/ref_mpp_drawcommand]] |
| `Private/MeshDrawCommands.cpp` | `FMeshDrawCommand::SubmitDraw`, `MatchesForDynamicInstancing`:1018 | PSO 適用・Draw 発行・インスタンシング判定 | [[Details/b_mpp_drawcommand]] |

### Shader・Binding

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `Public/MeshMaterialShader.h` | `FMeshMaterialShader`, `FMeshMaterialShaderType` | メッシュ用マテリアルシェーダー基底 | [[Reference/ref_mpp_shaders]] |
| `Public/MeshMaterialShaderType.h` | `FMeshMaterialShaderType::Permutation` | パーミュテーション定義 | 同 |
| `Public/MeshDrawShaderBindings.h` | `FMeshDrawShaderBindings`, `FMeshDrawShaderBindingsLayout` | シェーダーバインディングストレージ | [[Details/c_mpp_shaders]] |

### ユーティリティ

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `Public/MeshPassUtils.h` | `FMeshPassProcessorRenderState` ヘルパー | ブレンド / DepthStencil ステート | [[Reference/ref_mpp_utils]] |
| `Public/SimpleMeshDrawCommandPass.h` | `AddSimpleMeshPass()` | 単発パス用ヘルパー | 同 |
| `Private/MeshDrawCommandStats.h/cpp` | `FMeshDrawCommandStats` | デバッグ統計・ビジュアライザ | — |

---

## EMeshPass → 主要プロセッサ実装

`EMeshPass` の各値に対応する `FMeshPassProcessor` サブクラス（抜粋）。

| EMeshPass | プロセッサクラス | 実装ファイル | サブフォルダ |
|-----------|--------------|------------|-----------|
| `DepthPass` | `FDepthPassMeshProcessor` | `Private/DepthRendering.cpp` | [[DepthPrepass/]] |
| `BasePass` | `FBasePassMeshProcessor` | `Private/BasePassRendering.cpp` | [[BasePass/]] |
| `CSMShadowDepth` | `FShadowDepthPassMeshProcessor` | `Private/ShadowDepthRendering.cpp` | [[Shadow/]] |
| `VSMShadowDepth` | `FVirtualShadowMapMeshProcessor` | `Private/VirtualShadowMaps/` | [[VirtualShadowMaps/]] |
| `Velocity` | `FVelocityMeshProcessor` | `Private/VelocityRendering.cpp` | — |
| `TranslucencyStandard` | `FTranslucentMeshProcessor` | `Private/TranslucentRendering.cpp` | [[Translucency/]] |
| `CustomDepth` | `FCustomDepthPassMeshProcessor` | `Private/CustomDepthRendering.cpp` | — |
| `LumenCardCapture` | `FLumenCardMeshProcessor` | `Private/Lumen/LumenSurfaceCacheFeedback.cpp` | [[Lumen/]] |
| `NaniteMeshPass` | `FNaniteMeshProcessor` | `Private/Nanite/NaniteMaterials.cpp` | [[Nanite/]] |
| `MeshDecal_DBuffer` | `FMeshDecalsMeshProcessor` | `Private/MeshDecals.cpp` | [[Decals/]] |
| `HitProxy` | `FHitProxyMeshProcessor` | `Private/HitProxyRendering.cpp` | (Editor) |

---

## ライフサイクル → ファイル対応

### 静的コマンドキャッシュ（シーン追加時・1 回）

```
ENQUEUE_RENDER_COMMAND
  └─ FScene::AddPrimitive                           ScenePrivate.cpp
       └─ FPrimitiveSceneInfo::AddStaticMeshes      PrimitiveSceneInfo.cpp:1537
            └─ CacheMeshDrawCommands               PrimitiveSceneInfo.cpp:583
                 └─ EMeshPass 毎:
                    FPassProcessorManager::CreateMeshPassProcessor  MeshPassProcessor.h:2353
                      └─ AddMeshBatch             (各プロセッサ派生)
                           └─ BuildMeshDrawCommands MeshPassProcessor.h (template)
                                └─ FCachedPassMeshDrawListContextImmediate::FinalizeCommand
                                                    MeshPassProcessor.cpp:2032
```

### 毎フレーム（動的コマンド + Submit）

```
FSceneRenderer::BeginInitViews                     SceneVisibility.cpp
  ├─ FrustumCull / HZB                             → PrimitiveVisibilityMap
  ├─ ComputeRelevance                              → PrimitiveViewRelevanceMap
  └─ GatherDynamicMeshElements
       └─ AddMeshBatch (動的プリミティブ)
            └─ FDynamicPassMeshDrawListContext::FinalizeCommand  MeshPassProcessor.h:1820

FDeferredShadingSceneRenderer::RenderXxxPass()
  ├─ VisibleMeshDrawCommands に静的コマンドコピー
  ├─ SortMeshDrawCommands (FMeshDrawCommandSortKey 昇順)
  └─ SubmitMeshDrawCommands                        MeshPassProcessor.cpp:1604
       └─ SubmitMeshDrawCommandsRange              MeshPassProcessor.cpp:1616
            └─ FMeshDrawCommand::SubmitDraw
                 ├─ SubmitDrawBegin: PSO セット
                 └─ SubmitDrawEnd:   DrawIndexedPrimitive
```

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_mpp_processor]] | FMeshPassProcessor 基底 |
| Reference | [[Reference/ref_mpp_drawcommand]] | FMeshDrawCommand |
| Reference | [[Reference/ref_mpp_shaders]] | FMeshMaterialShader |
| Reference | [[Reference/ref_mpp_utils]] | RenderState ヘルパー |
| Details | [[Details/a_mpp_pipeline]] | パイプライン全体 |
| Details | [[Details/b_mpp_drawcommand]] | DrawCommand 生成詳細 |
| Details | [[Details/c_mpp_shaders]] | シェーダーバインディング |
| Details | [[Details/d_mpp_passes]] | EMeshPass パス別解説 |

---

## ue5-dive 起点

- 「新しい EMeshPass を追加するには」 → `MeshPassProcessor.h:EMeshPass` + 新規 FXxxMeshProcessor 派生
- 「Static/Dynamic の境目」 → `PrimitiveSceneInfo.cpp:CacheMeshDrawCommands` と `GatherDynamicMeshElements`
- 「PSO 切り替え最小化の仕組み」 → `MeshDrawCommands.cpp:FMeshDrawCommandStateCache`
- 「動的インスタンシング条件」 → `MeshDrawCommands.cpp:MatchesForDynamicInstancing:1018`
