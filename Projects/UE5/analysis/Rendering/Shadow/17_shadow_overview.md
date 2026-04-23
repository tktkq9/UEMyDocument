# Shadow Rendering 全体概要

- 取得日: 2026-04-12
- 対象: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\ShadowRendering.h/.cpp` / `ShadowSetup.cpp`
- 上位: [[01_rendering_overview]]
- Details: [[a_shadow_setup]] | [[b_shadow_depth]] | [[c_shadow_projection]]
- Reference: [[ref_projected_shadow_info]] | [[ref_shadow_rendering]] | [[ref_shadow_projection]]
- 注意: **VSM（Virtual Shadow Maps）は別フォルダ** `VirtualShadowMaps/` を参照

---

## Shadow システムとは

UE5 の非 VSM シャドウは **FProjectedShadowInfo** を中心に管理される。  
ライトごと・プリミティブごとに Shadow Depth Map を描画し、  
Lighting Pass で PCF / PCSS によるソフトシャドウとして SceneColor に適用する。

| シャドウ種別 | 対象ライスト | 説明 |
|-----------|----------|------|
| Whole Scene Shadow / CSM | Directional Light | カスケード分割（1〜4枚）の Shadow Map |
| Spot Light Shadow | Spot Light | コーン錐台の Shadow Map |
| Point Light Shadow | Point Light | キューブマップ Shadow（6面）|
| Per-Object Shadow | 動的オブジェクト | プリミティブごとの個別 Shadow |
| Cached Shadow | Stationary Light | 静的ジオメトリのシャドウをキャッシュ |

---

## 全体アーキテクチャ

```mermaid
graph TD
    subgraph Setup["SetupMeshDrawCommands（InitViews）"]
        SI[ShadowSetup.cpp<br>FProjectedShadowInfo を生成・分類<br>種別 / アトラス位置 / カスケード設定]
    end

    subgraph DepthPass["Shadow Depth 描画"]
        SD[RenderShadowDepthMaps<br>FShadowDepthMeshProcessor → DrawCall<br>Shadow Depth テクスチャに書き込み]
        NSD[Nanite Shadow<br>Nanite::RenderShadowDepths]
    end

    subgraph Projection["Shadow Projection"]
        SP[RenderProjectedShadow<br>FShadowProjectionPS（PCF/PCSS）<br>→ Shadow Mask テクスチャ生成]
    end

    subgraph Apply["ライストへの適用"]
        RL[RenderLight()<br>ShadowMaskTexture をバインド<br>ライスト寄与に乗算]
    end

    Setup --> DepthPass --> Projection --> Apply
```

---

## FProjectedShadowInfo の生存期間

```
InitViews()
  └─ SetupMeshDrawCommandsForShadowDepth()
      → FProjectedShadowInfo を生成（種別ごと）
      → FVisibleLightInfo::AllProjectedShadows に追加

RenderShadowDepthMaps()
  → FProjectedShadowInfo::RenderDepth()
      → Shadow Depth テクスチャに書き込み

RenderLights()
  → for each UnbatchedLight:
      FProjectedShadowInfo::RenderProjectedShadow()
          → Shadow Mask テクスチャ生成
      → RenderLight(..., ShadowMaskTexture)
```

---

## FShadowCascadeSettings（CSM カスケード設定）

```cpp
// ShadowRendering.h（概略）
struct FShadowCascadeSettings
{
    // カスケードの分割距離（View Space Z）
    float SplitNear;          // 近平面
    float SplitFar;           // 遠平面
    float SplitNearFadeRegion;// フェード開始
    float SplitFarFadeRegion; // フェード終端
    // カスケード間のブレンド比率
    float FadePlaneOffset;
    float FadePlaneLength;
    // カスケードの投影方向（Whole Scene Shadow）
    int32 ShadowSplitIndex;   // 0〜3
};
// r.Shadow.CSM.MaxCascades（デフォルト 4）で枚数を制限
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Shadow.CSM.MaxCascades` | 4 | CSM カスケード最大枚数 |
| `r.Shadow.MaxResolution` | 2048 | Shadow Map の最大解像度 |
| `r.Shadow.CSM.TransitionScale` | 1.0 | CSM カスケード間のフェード幅 |
| `r.Shadow.DistanceScale` | 1.0 | Shadow 描画距離スケール |
| `r.Shadow.PerObject` | 1 | Per-Object Shadow の有効/無効 |
| `r.Shadow.CacheWholeSceneShadows` | 1 | 静的シャドウキャッシュの有効化 |
| `r.Shadow.FilterMethod` | 1 | フィルタ方式（0=PCF/1=PCSS）|

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `ShadowRendering.h/.cpp` | `FProjectedShadowInfo`・Shadow Projection パス |
| `ShadowSetup.cpp` | シャドウ種別分類・アトラスパッキング |
| `ShadowDepthRendering.h/.cpp` | Shadow Depth Map 描画・`FShadowDepthMeshProcessor` |
| `ShadowProjectionPixelShader.h` | `FShadowProjectionPS` テンプレート群 |
| `Shadows/ShadowSceneRenderer.h/.cpp` | UE5.4+ の Shadow Scene 管理（VSM 連携）|

---

## RenderShadowDepthMaps() → Projection 詳細フロー

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─ [InitViews フェーズ]
  │   SetupMeshDrawCommandsForShadowDepth()    ShadowSetup.cpp
  │     → for each FLightSceneInfo:
  │         CreatePerObjectProjectedShadow() または CreateWholeSceneProjectedShadow()
  │           → FProjectedShadowInfo を生成
  │           → Atlas 上の (X, Y, ResolutionX, ResolutionY) を割り当て
  │           → FVisibleLightInfo::AllProjectedShadows に追加
  │
  ├─ RenderShadowDepthMaps()                   ShadowRendering.cpp
  │   for each FProjectedShadowInfo:
  │     ├─ [Nanite] Nanite::RenderShadowDepths() → CS で深度書き込み
  │     └─ [非Nanite] FShadowDepthMeshProcessor → DrawCall
  │         FShadowDepthVS で Shadow View Matrix 変換
  │         （Masked: FShadowDepthPS で clip）
  │         Shadow Depth Atlas テクスチャ（R32_FLOAT or D24）に書き込み
  │
  └─ RenderLights() 内 UnbatchedLights ループ
      FProjectedShadowInfo::RenderProjectedShadow()  ShadowRendering.cpp
        ├─ TShadowProjectionPS<PCFSamples> でフルスクリーン or 境界ジオメトリを描画
        │   GBuffer.Depth と Shadow Depth を比較 → Shadow Factor
        │   PCF: 近傍サンプルの平均でソフトシャドウ
        │   PCSS: 遮蔽物サイズに応じてサンプル半径を変化
        ├─ Shadow Mask テクスチャ（R8）に書き込み
        └─ RenderLight(..., ShadowMaskTexture)
            → DeferredLightPixelShaders.usf で乗算適用
```
