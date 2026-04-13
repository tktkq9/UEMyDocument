# Ray Tracing 全体概要

- 取得日: 2026-04-10
- 対象: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\RayTracing\`
- 上位: [[01_rendering_overview]]

---

## Ray Tracing とは

DXR（DirectX 12 Ultimate）/ Vulkan Ray Tracing 対応時にのみ有効になる  
**HW アクセラレーションレイトレーシング基盤**。  
Lumen の HW RT バックエンドとして使われるほか、  
AO・シャドウ・反射・スカイライト等の独立した品質優先モードも提供する。

| 機能 | 説明 |
|------|------|
| Lumen HW RT | Lumen のトレースバックエンドとして透過的に使用 |
| RT シャドウ | ディレクショナル・スポット等の高品質シャドウ |
| RT AO | ハードウェアレイによるアンビエントオクルージョン |
| RT リフレクション | 鏡面反射の高品質モード |
| RT スカイライト | スカイライトのシャドウイング |
| MegaLights RT | MegaLights のシャドウバックエンド |

> **注意**: このフォルダは `RHI_RAYTRACING` マクロで全体がガードされており、  
> DXR/Vulkan RT 非対応ハードウェアではコンパイルされない。

---

## 全体アーキテクチャ

```mermaid
graph TD
    subgraph TLAS["TLAS（Top Level Acceleration Structure）"]
        RS[FRayTracingScene\nフレームごとのインスタンスリスト]
        GI[GatherInstances\nBLAS参照→インスタンス登録]
        RS --> GI
    end

    subgraph BLAS["BLAS（Bottom Level AS）"]
        SM[Static Mesh BLAS\n静的メッシュ]
        DY[Dynamic BLAS\nスケルタル等]
        NB[Nanite BLAS\nNaniteRayTracing経由]
    end

    subgraph SBT["FRayTracingShaderBindingTable"]
        HG[Hit Group シェーダー\nマテリアル別]
        MS[Miss シェーダー]
    end

    subgraph Passes["RT パス"]
        AO[RayTracingAmbientOcclusion]
        SH[RayTracingShadows]
        SL[RayTracingSkylight]
        PR[RayTracingPrimaryRays]
        TR[RayTracingTranslucency]
        DC[RayTracingDecals]
    end

    GI --> RS
    SM & DY & NB --> TLAS
    TLAS --> SBT --> Passes
```

---

## フレームの流れ（概略）

```
[A] RayTracing::OnRenderBegin()
    → フレーム開始時のシーン更新

[B] CreateGatherInstancesTaskData() / AddView()
    → ビューごとのインスタンス収集タスクを作成

[C] BeginGatherInstances()
    → 並列タスクで各プリミティブ BLAS を TLAS に登録
    → フラスタムカリングタスクと依存関係を設定

[D] BeginGatherDynamicRayTracingInstances()
    → スケルタルメッシュ等の動的 BLAS 更新

[E] FinishGatherInstances()
    → TLAS の構築コマンドを RDG に発行

[F] FinishGatherVisibleShaderBindings()
    → 可視インスタンスのヒットグループシェーダーバインドを確定

[G] 各 RT パス
    → RayTracingShadows / AO / Skylight 等を独立して実行
    → Lumen から呼ばれる HW RT トレースも同じ TLAS を使用
```

---

## 主要クラス・構造体

```cpp
// シーンオプション（RT インスタンス収集時の設定）
namespace RayTracing
{
    struct FSceneOptions
    {
        bool bTranslucentGeometry;     // 半透明ジオメトリを含めるか
        bool bIncludeSky;              // スカイを含めるか
        bool bLightingChannelsUsingAHS;// ライティングチャンネルを AHS で処理するか
    };

    // インスタンス収集タスクのデータ（並列処理用）
    struct FGatherInstancesTaskData;
}

// RT シーン（フレームごとに構築）
class FRayTracingScene
{
    // TLAS に登録するインスタンスのリスト
    TArray<FRayTracingGeometryInstance> Instances;
    // TLAS バッファ（RDG）
    FRDGBufferRef TLASBuffer;
};

// シェーダーバインディングテーブル
class FRayTracingShaderBindingTable
{
    // ヒットグループごとのシェーダーバインドデータ
    TArray<FRayTracingShaderBindingData> VisibleShaderBindings;
};
```

---

## 主要 CVar 一覧

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RayTracing` | 0 | RT 全体の有効/無効（プロジェクト設定） |
| `r.RayTracing.Shadows` | -1 | RT シャドウ（-1=自動, 0=無効, 1=有効） |
| `r.RayTracing.AmbientOcclusion` | -1 | RT AO |
| `r.RayTracing.Reflections` | -1 | RT リフレクション |
| `r.RayTracing.SkyLight` | 1 | RT スカイライト |
| `r.RayTracing.ExcludeDecals` | 1 | デカールを TLAS から除外 |
| `r.RayTracing.InstanceCulling` | 1 | フラスタムカリング有効 |

---

## 主要ソースファイル一覧

| ファイル | 役割 |
|---------|------|
| `RayTracing.h/.cpp` | インスタンス収集・TLAS 構築・全体エントリ |
| `RayTracingScene.h/.cpp` | FRayTracingScene（TLAS）管理 |
| `RayTracingShaderBindingTable.h/.cpp` | SBT（ヒットグループシェーダーバインド）管理 |
| `RayTracingInstanceCulling.h/.cpp` | フラスタムカリング・インスタンスフィルタリング |
| `RayTracingInstanceMask.h/.cpp` | インスタンスマスク（ライティングチャンネル） |
| `RayTracingMaterialHitShaders.h/.cpp` | マテリアルヒットシェーダーのコンパイル・登録 |
| `RayTracingShadows.h/.cpp` | RT シャドウパス |
| `RayTracingAmbientOcclusion.cpp` | RT AO パス |
| `RayTracingSkyLight.h/.cpp` | RT スカイライトパス |
| `RayTracingLighting.h/.cpp` | RT ライティング共通ユーティリティ |
| `RayTracingPrimaryRays.cpp` | 1次レイ（反射等の高品質モード） |
| `RayTracingTranslucency.cpp` | 半透明 RT |
| `RayTracingDecals.h/.cpp` | デカールの RT 対応 |
| `RayTracingDynamicGeometryUpdateManager.cpp` | 動的ジオメトリの BLAS 更新管理 |
| `RaytracingOptions.h` | RT オプション構造体の定義 |

---

## コード実行フロー

### エントリポイント

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─ [1] RayTracing::OnRenderBegin()
  │       フレーム開始時の RT シーン状態リセット
  │
  ├─ [2] RayTracing::CreateGatherInstancesTaskData()
  │   RayTracing::AddView()
  │       ビューごとの FGatherInstancesTaskData / FSceneOptions を初期化
  │
  ├─ [3] RayTracing::BeginGatherInstances()          // 並列タスク開始
  │       └─ FrustumCullTask 完了後に各 BLAS → TLAS 登録
  │           ├─ RayTracing::CullPrimitiveByFlags()
  │           └─ RayTracing::ShouldCullBounds()
  │
  ├─ [4] RayTracing::BeginGatherDynamicRayTracingInstances()
  │       スケルタルメッシュ等の動的 BLAS を非同期更新
  │
  ├─ [5] RayTracing::FinishGatherInstances()         // レンダースレッド
  │       ├─ FRayTracingScene::Update()              // GPU インスタンスバッファ転送
  │       └─ FRayTracingScene::Build()               // TLAS 構築 RDG パス
  │
  ├─ [6] RayTracing::FinishGatherVisibleShaderBindings()
  │       └─ AddRayTracingLocalShaderBindingWriterTasks()  // 並列バインド書き込み
  │           └─ SetRayTracingShaderBindings()              // RHI 反映
  │
  ├─ [7] RayTracingShadows::RenderRayTracingShadows()      // 光源ごと
  │       └─ IScreenSpaceDenoiser::Denoise()
  │
  ├─ [8] RenderRayTracingAmbientOcclusion()                // 有効時のみ
  │       └─ GScreenSpaceDenoiser->DenoiseAmbientOcclusion()
  │
  ├─ [9] RenderRayTracingSkyLight()                        // r.RayTracing.SkyLight=1
  │
  ├─ [10] RenderRayTracingReflections()                    // 有効時のみ
  │        └─ IScreenSpaceDenoiser::DenoiseReflections()
  │
  └─ [11] RenderRayTracingTranslucency()                   // 有効時のみ
```

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|------------|--------|------|
| `RayTracing::OnRenderBegin()` | `RayTracing.cpp` | フレーム開始リセット |
| `RayTracing::FGatherInstancesTaskData` | `RayTracing.cpp` | 並列タスクデータホルダー |
| `FRayTracingScene::Update()` | `RayTracingScene.cpp` | GPU バッファ転送 |
| `FRayTracingScene::Build()` | `RayTracingScene.cpp` | TLAS RDG パス |
| `RayTracing::FinishGatherVisibleShaderBindings()` | `RayTracing.cpp` | SBT バインド確定 |
| `SetRayTracingShaderBindings()` | `RayTracingMaterialHitShaders.h` | RHI 反映 |

### サブシステムドキュメント

| ドキュメント | 内容 |
|------------|------|
| [[a_rt_scene]] | TLAS / BLAS 構築・GatherInstances・フラスタムカリング |
| [[b_rt_shadow_ao]] | RT シャドウ・RT AO・RT スカイライト |
| [[c_rt_reflection]] | RT リフレクション・RT 半透明 |
| [[d_rt_materials_sbt]] | マテリアルヒットシェーダー・SBT 管理 |
