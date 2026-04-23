# Base Pass（GBuffer）全体概要

- 取得日: 2026-04-12
- 対象: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\BasePassRendering.h/.cpp`
- 上位: [[01_rendering_overview]]
- Details: [[a_basepass_pipeline]] | [[b_gbuffer_layout]] | [[c_basepass_material]]
- Reference: [[ref_basepass_renderer]] | [[ref_basepass_common]] | [[ref_gbuffer_textures]]
- GPU 対応: [[GPU/BasePass/TASK_CHECKLIST]]

---

## Base Pass とは

**不透明マテリアルのシェーディング結果を GBuffer（Geometry Buffer）に書き込むパス**。  
Deferred Rendering の核心部分であり、このパスで生成された GBuffer データを使って  
後続の Lighting Pass が照明計算を行う。

| 担当 | 内容 |
|------|------|
| BasePass | GBuffer 書き込み（マテリアル評価） |
| LightingPass | GBuffer 読み取り + 照明計算 |
| Nanite | GBuffer Resolve（VisBuffer → GBuffer 変換）|

---

## 全体アーキテクチャ

```mermaid
graph TD
    subgraph Input["入力"]
        Prims[可視プリミティブ一覧<br>FViewInfo::VisibleMeshDrawCommands]
        DBuffer[DBuffer デカール<br>FDBufferTextures]
    end

    subgraph MeshProcessor["TBasePassMeshProcessor"]
        AddMesh[AddMeshBatch()<br>マテリアル × ライトマップ種別で<br>シェーダー排列を生成]
    end

    subgraph RDG["RDG Pass"]
        Parallel[並列描画<br>r.ParallelBasePass=1]
        Serial[逐次描画]
    end

    subgraph Output["GBuffer 出力"]
        GBA[GBufferA: WorldNormal]
        GBB[GBufferB: Metallic/Spec/Rough/ShadingModelID]
        GBC[GBufferC: BaseColor + AO]
        GBD[GBufferD: CustomData]
        Vel[Velocity Buffer]
    end

    Input --> MeshProcessor --> RDG --> Output
```

---

## フレームの流れ（概略）

```
FDeferredShadingSceneRenderer::Render()
  │
  └─ RenderBasePass()                          BasePassRendering.cpp:1071
      │
      ├─ GBufferクリア（r.ClearSceneMethod）
      ├─ Substrate ステンシルマーク（Substrate 有効時）
      ├─ GetGBufferRenderTargets()              GBuffer テクスチャを RT に束ねる
      │
      ├─ [bNaniteEnabled == true の場合]
      │   └─ NaniteShading::EmitGBuffer()      VisBuffer → GBuffer 変換
      │
      └─ RenderBasePassInternal()              非 Nanite メッシュの描画
          ├─ [r.ParallelBasePass=1] ParallelCommandList
          │   └─ TBasePassMeshProcessor::AddMeshBatch() × 各プリミティブ
          └─ [r.ParallelBasePass=0] 逐次描画
```

---

## GBuffer レイアウト（従来 Shading Model）

| スロット | フォーマット | 内容 |
|--------|------------|------|
| GBufferA | R8G8B8A8 | WorldNormal(RGB) + PerObjectGBufferData(A) |
| GBufferB | R8G8B8A8 | Metallic(R) + Specular(G) + Roughness(B) + ShadingModelID + SelectiveMask(A) |
| GBufferC | R8G8B8A8 | BaseColor(RGB) + IndirectIrradiance / AO(A) |
| GBufferD | R8G8B8A8 | CustomData（ShadingModel 依存） |
| GBufferE | R8G8B8A8 | Pre-computed shadow factor（Stationary Light） |
| GBufferF | R8G8B8A8 | WorldTangent(RGB) + Anisotropy(A)（Anisotropy 有効時）|
| Velocity | R16G16 | Motion Vector（TSR/TAA/MotionBlur 用）|
| SceneDepth | D24S8 | 深度 + Stencil（ShadingModel ID） |

> **Substrate 有効時**: MaterialTextureArray（Texture2DArray uint）に置き換わる。  
> スライス数は `r.Substrate.BytesPerPixel` によって決まる。

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.ParallelBasePass` | 1 | 並列 BasePass 描画の有効/無効 |
| `r.SelectiveBasePassOutputs` | 0 | 関連 RT のみへの Export（シェーダーリコンパイル必要）|
| `r.ClearGBufferDBeforeBasePass` | 1 | GBufferD を BasePass 前にクリア |
| `r.ClearSceneMethod` | 1 | GBuffer クリア方法（0=なし / 1=RHIClear / 2=Far-Z Quad）|
| `r.AllowGlobalClipPlane` | 0 | Planar Reflection 用グローバルクリッププレーン（シェーダーリコンパイル必要）|

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `BasePassRendering.h/.cpp` | `RenderBasePass()` / `TBasePassMeshProcessor` |
| `BasePassRendering.inl` | テンプレートシェーダー実装（`TBasePassVS/PS`）|
| `BasePassCommon.ush` | GBuffer 共通マクロ・パラメータ定義 |
| `SceneTextures.h/.cpp` | `FSceneTextures`（GBuffer テクスチャ束の管理）|
| `GBufferInfo.h` | `FGBufferData` デコード構造体 |
| `Nanite/NaniteShading.h/.cpp` | NaniteEmitGBuffer / ShadeBinning |

---

## RenderBasePass() → MeshPassProcessor → RDG Pass 発行 詳細フロー

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─ [事前] InitViews()
  │   └─ ComputeViewVisibility() → VisibleMeshDrawCommands をビューごとに収集
  │
  └─ RenderBasePass()                             BasePassRendering.cpp:1071
      │
      ├─ [1] GBuffer テクスチャの確保
      │   SceneTextures.GetGBufferRenderTargets()
      │   → GBufferA/B/C/D/E/F + SceneColor + Velocity を
      │     TStaticArray<FTextureRenderTargetBinding> に束ねる
      │
      ├─ [2] UniformBuffer 構築
      │   CreateOpaqueBasePassUniformBuffer()
      │   → FOpaqueBasePassUniformParameters に
      │     ForwardLight / Reflection / Fog / DBuffer / EyeAdaptation を詰める
      │
      ├─ [3] Nanite EmitGBuffer（Nanite メッシュ）
      │   NaniteShading::EmitGBuffer()
      │   → VisBuffer（uint64: ClusterID + TriangleID）を読んで
      │     NaniteBasePassShadingCommands を CS でディスパッチ
      │   → 通常 Draw Call を一切発行せず直接 GBuffer に書き込む
      │
      └─ [4] RenderBasePassInternal()              BasePassRendering.cpp:1449
          │
          ├─ RDG RenderPass 開始（GBuffer RT バインド・Depth R/W）
          │
          ├─ [r.ParallelBasePass=1]
          │   FParallelCommandListSet set(..., Views.Num())
          │   for each View:
          │     ParallelCommandList.Execute(
          │       DrawDynamicMeshPass(View, TBasePassMeshProcessor, ...))
          │
          └─ [r.ParallelBasePass=0] 逐次
              for each View:
                DrawDynamicMeshPass(View, TBasePassMeshProcessor, ...)
                  └─ per FMeshBatch:
                      TBasePassMeshProcessor::AddMeshBatch()
                        → GetUniformLightMapPolicy() でポリシー決定
                        → TBasePassPS<Policy,NumDynPtLights> 取得
                        → BuildMeshDrawCommands() → Submit
```

> [!note]- RDG の Cull による省略
> RDG は未参照テクスチャへの Pass を自動でカリングする。
> `SelectiveBasePassOutputs` が有効な場合は Unlit マテリアルが GBufferA/B への
> 書き込みをスキップするため、ライティングパスが GBufferB を参照しない場合は
> そのテクスチャ書き込み Pass 自体が RDG に除去される場合がある。
