---
name: Lumen Surface Cache
description: カードベース放射輝度キャッシュ（Lumen GI / Reflections / Radiosity 全てが読む共通入力）
type: project
---

# Lumen Surface Cache（カードベース放射輝度キャッシュ）

- 上位: [[_algorithm_index]]
- 関連: [[lumen_radiance_cache]] / [[lumen_sw_rt]] / [[lumen_hw_rt]] / [[lumen_final_gather]]
- 採用システム: Lumen GI（ScreenProbeGather, ReSTIR Gather, Reflections, Radiosity 全てが SurfaceCache を読む）
- 出典 ID: **S10**（[[_source_index]]）— §3 Surface Cache
- 補助資料: Epic 公式 "Lumen Technical Details"（UE 公式 Docs）

---

## 1. 何のためのアルゴリズムか

レイトレース時に **ヒット点で完全な BasePass + DirectLighting を再評価する代わりに、事前に計算した放射輝度をテクスチャから読むだけ** にするキャッシュ機構。

### 素朴な手法の問題

- **Path Tracing 流**: ヒットごとに material 評価 + shadow ray + light evaluation → 1 ray あたりのコストが膨大、リアルタイム不可
- **Voxel Cone Tracing (VXGI)**: ボクセル内の平均 → 細部消失、表面沿いの bleeding
- **Irradiance Volume / Light Probe**: 空間離散化のみ、表面上の解像度が一定で偏る

### Lumen Surface Cache の貢献

- **メッシュごとに「カード」と呼ばれる 2D 矩形パッチ群を貼り付け**、各カードに G-Buffer 風の Depth/Albedo/Normal/Emissive をベイク（**ベイクではなく毎フレーム capture**）
- カードは **Mesh Cards 表現**（オフライン生成の AABB 6 面 ± マージング）
- カード上に **DirectLighting（ShadowMap 流）** + **Radiosity（多重バウンス）** を毎フレーム積算 → **FinalLighting アトラス**
- レイヒット時はメッシュ ID + カード ID + UV から **テクスチャ 1 サンプリング** で放射輝度取得 → 劇的な高速化

---

## 2. 理論

### 2.1 Mesh Cards 表現

- メッシュの bounding box を起点に、各軸 ± 6 方向の正射影矩形を生成
- カードごとに表面が「写る」面積を評価し、寄与の小さいカードを除外（`r.LumenScene.SurfaceCache.MeshCardsMinSize` = 10cm）
- カードの解像度は距離・サーフェス面積・スクリーン投影サイズに応じて 8×8〜2048×2048（`MinResLevel=3`〜`MaxResLevel=11`）

### 2.2 アトラス階層

カードは 4 階層のアトラスに統合:

| アトラス | 内容 | 用途 |
|---------|------|------|
| **Depth Atlas** | 16-bit Depth | レイヒット時の SDF / カード変換、再投影検証 |
| **Albedo Atlas** | BC7（圧縮時）/ R8G8B8A8 | DiffuseColor |
| **Normal Atlas** | BC5 / R8G8 | 法線（接線空間圧縮） |
| **Emissive Atlas** | BC6H / FloatR11G11B10 | Emissive HDR |
| **Opacity Atlas** | G8 | アルファマスク |
| **DirectLighting Atlas** | R11G11B10 等 | Direct + Skylight + Shadowing |
| **IndirectLighting / FinalLighting Atlas** | R11G11B10 等 | Radiosity 結果（多重バウンス） |

### 2.3 ストリーミング（Page-based）

- 全カードを GPU に常駐させると VRAM が爆発するため、**128×128 の物理ページ** で分割
- 仮想ページ（127×127、0.5 texel border）→ 物理ページにマップ
- 各カードはスクリーン投影サイズに応じて適切な **ResLevel** を選択し、必要なページのみアロケート
- フィードバック（`LumenSurfaceCacheFeedback.cpp`）: 実際にレイがヒットしたページを GPU 側でマーク → CPU が次フレームのストリーミング判断

### 2.4 Capture（毎フレーム微更新）

- 全シーンを毎フレーム再キャプチャするのは重い
- **更新頻度**: カードごとに「最後に capture された時刻」を保持、距離・優先度に応じて N フレームに 1 回だけ MeshCard 全体を再描画
- Capture は **LumenCardVertexShader.usf / LumenCardPixelShader.usf** で MeshDrawProcessor を回し、Albedo/Normal/Emissive を書く（BasePass 簡易版）

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `LumenMeshCards.cpp` | カード代表生成、merge instance 管理 |
| `LumenSceneCardCapture.cpp` | カードへの BasePass capture（VS/PS パス） |
| `LumenSurfaceCache.cpp` | アトラス管理、レイアウト、圧縮、Dilation |
| `LumenSurfaceCacheFeedback.cpp` | GPU→CPU フィードバックでページ優先度算出 |
| `LumenSceneLighting.cpp` | DirectLighting アトラス更新 |
| `LumenRadiosity.cpp` | IndirectLighting（多重バウンス）アトラス更新 |

### 3.2 アトラス設定（`LumenSurfaceCache.cpp:100-114`）

```cpp
const FLumenSurfaceLayerConfig& GetSurfaceLayerConfig(ELumenSurfaceCacheLayer Layer)
{
    static FLumenSurfaceLayerConfig Configs[(uint32)ELumenSurfaceCacheLayer::MAX] =
    {
        { TEXT("Depth"),    PF_G16,            PF_Unknown, PF_Unknown,            FVector(1, 0, 0) },
        { TEXT("Albedo"),   PF_R8G8B8A8,       PF_BC7,     PF_R32G32B32A32_UINT,  FVector(0, 0, 0) },
        { TEXT("Opacity"),  PF_G8,             PF_Unknown, PF_Unknown,            FVector(1, 0, 0) },
        { TEXT("Normal"),   PF_R8G8,           PF_BC5,     PF_R32G32B32A32_UINT,  FVector(0, 0, 0) },
        { TEXT("Emissive"), PF_FloatR11G11B10, PF_BC6H,    PF_R32G32B32A32_UINT,  FVector(0, 0, 0) }
    };
    ...
}
```

UAV エイリアシング（`PF_R32G32B32A32_UINT` ↔ BC7）で **GPU 側でリアルタイム BC7 圧縮** を実現。

### 3.3 物理ページ定数（`Lumen.h:38-46`）

```cpp
namespace Lumen
{
    constexpr uint32 PhysicalPageSize = 128;
    constexpr uint32 VirtualPageSize = PhysicalPageSize - 1; // 0.5 texel border around page
    constexpr uint32 MinCardResolution = 8;
    constexpr uint32 MinResLevel = 3;   // 2^3 = MinCardResolution
    constexpr uint32 MaxResLevel = 11;  // 2^11 = 2048 texels
    constexpr uint32 SubAllocationResLevel = 7; // log2(PHYSICAL_PAGE_SIZE)
    ...
}
```

`VirtualPageSize = 127` の **0.5 texel border** はバイリニアサンプリング時の隣接ページからの leak 防止。

### 3.4 LOD と Capture 解像度（`LumenSceneCardCapture.cpp:18-46`）

```cpp
static TAutoConsoleVariable<float> GLumenSceneSurfaceCacheMeshTargetScreenSize(
    TEXT("r.LumenScene.SurfaceCache.MeshTargetScreenSize"),
    0.15f,  // 画面の 15% を占める時のメッシュサイズで LOD 選択
    ...);

static TAutoConsoleVariable<float> GLumenSceneSurfaceCacheNaniteLODScaleFactor(
    TEXT("r.LumenScene.SurfaceCache.NaniteLODScaleFactor"),
    1.0f,  // Nanite メッシュは別途スケール
    ...);
```

Nanite メッシュは Capture 時に低 LOD クラスタを指定（`NaniteLODScaleFactor`）。

### 3.5 Mesh Cards Merging（`LumenMeshCards.cpp:22-92`）

- `r.LumenScene.SurfaceCache.MeshCardsMergeComponents = 1`: 同じ `RayTracingGroupId` のコンポーネントを 1 つの MeshCards に統合
- `r.LumenScene.SurfaceCache.MeshCardsMergedResolutionScale = 0.3`: マージ後カードの解像度を 30% に下げる（細部不要）
- 目的: ストリーミング単位（オブジェクト粒度）と GPU dispatch コストの削減

### 3.6 Dilation（`LumenSurfaceCache.cpp:14-25`）

```cpp
// 0 - Disabled
// 1 - Only two-sided (foliage but currently only wired up for mostly two-sided Nanite skinned meshes)
// 2 - All
```

カード境界外側 1 texel に色を伸ばす。Foliage 等の two-sided マテリアルでカードがメッシュ表面と少しずれている場合のヒット失敗を救済。

---

## 4. 近似・省略の差分

| 項目 | 理想（Path Tracing） | Lumen Surface Cache | 影響 |
|------|------------------|------------------|------|
| ヒット時の評価 | 完全 material 評価 + shadow ray | アトラス 1 sample | ~100x 高速化 |
| 解像度 | 連続 | カード ResLevel 8〜2048 | 細部 detail loss、Hit Lighting モードで補完可能 |
| カード方向 | 完全（任意法線） | 6 軸 ± + マージ | 法線急変箇所で誤マッピング |
| 透過 | 完全 | Opacity 1 ch のみ | 半透明は別経路（Translucency Volume） |
| 動的更新 | リアルタイム | N フレームに 1 回 | 動く光源で 1〜数フレーム遅延 |
| サーフェス無し領域 | 自然に処理 | Mesh Cards にカードが無い → ヒット失敗 → 黒 or fallback | Foliage / 細い枝で問題、Dilation で軽減 |

**Hit Lighting モード**（`r.Lumen.HardwareRayTracing.LightingMode = 1 or 2`）で完全評価に切替可能だが GPU コスト数倍。

---

## 5. パラメータと CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.LumenScene.SurfaceCache.MeshTargetScreenSize` | 0.15 | LOD 選択基準（画面占有比率） |
| `r.LumenScene.SurfaceCache.MeshCardsMinSize` | 10.0 cm | このサイズ未満のカードは生成しない |
| `r.LumenScene.SurfaceCache.MeshCardsMergeComponents` | 1 | RayTracingGroupId でマージ |
| `r.LumenScene.SurfaceCache.Compress` | 1 | 0=無圧縮 / 1=UAV エイリアス BC / 2=CopyTexture |
| `r.LumenScene.SurfaceCache.DilationMode` | 0 | 0/1/2（Disable/two-sided/All） |
| `r.LumenScene.SurfaceCache.CullUndergroundTexels` | 0 | 地中ピクセル除去（要 RVT） |

---

## 6. 代替手法との比較

| 手法 | キャッシュ単位 | レイ評価コスト | 動的更新 | 採用 |
|------|------------|------------|--------|------|
| Path Tracing（参照） | なし | 重 | 即時 | Lumen Path Tracer モード |
| Voxel Cone Tracing (VXGI) | ボクセル | 軽 | 速い | NVIDIA VXGI（UE 未統合） |
| Irradiance Volume | 3D グリッドプローブ | 軽 | 中 | UE Static Lighting / SkyLight |
| DDGI / RTXGI | 3D 球面プローブ | 中 | 中 | NVIDIA RTXGI |
| **Lumen Surface Cache** | **メッシュ表面 2D アトラス** | **軽** | **N フレーム遅延** | **UE5 標準** |
| ReSTIR （直接） | なし、再利用統計 | 重い初期 | 即時 | Lumen ReSTIR Gather と併用 |

### Lumen Surface Cache の独自性

- **表面に密着したキャッシュ** → 接触陰やコンタクトシャドウ的な near-field GI が表現できる（Voxel/Volume 系の弱点）
- **Mesh Cards 表現** はオフライン生成だが圧倒的に軽量
- 欠点: 形状が極端に複雑（葉、髪、配管）だと Cards でカバーしきれない → Hit Lighting fallback

---

## 7. 参考資料

- [S10] Wright / Heitz / Hillaire 2022, "Lumen: Real-time Global Illumination in Unreal Engine 5", SIGGRAPH 2022 Course — §3 Surface Cache 詳細
- Epic 公式: "Lumen Technical Details" / "Lumen Performance Guide"
- 関連: [[lumen_radiance_cache]]（遠方は Radiance Cache に切替）

---

## 8. 相談用フック

- **理解度チェック**:
  - なぜ「カード」と呼ぶのか → §2.1、メッシュ AABB の 6 軸正射影が「カード」
  - Surface Cache と Radiance Cache の役割分担 → Surface = 表面、Radiance = 空間プローブ（[[lumen_radiance_cache]]）
  - Hit Lighting モードを使う場面 → §4、Surface Cache が貧弱な細部メッシュ・反射
- **コード深掘り候補**:
  - `LumenMeshCards.cpp` の Mesh Cards 生成 (`MeshCardRepresentation`)
  - `LumenSurfaceCacheFeedback.cpp` の GPU フィードバックループ（実際にレイがヒットしたページ追跡）
  - UAV エイリアシングによる BC7 圧縮の実装（`LumenSurfaceCache.cpp` の `ESurfaceCacheCompression::UAVAliasing`）
- **未読箇所**:
  - S10 §3.4 Radiosity の収束判定
  - Mesh Cards のオフライン生成パス（Editor 側 `MeshCardBuild.cpp`）
- **次の派生**:
  - Radiosity 多重バウンス → [[lumen_radiosity]]（未着手）
  - Direct Lighting アトラスのシャドウ手法 → [[lumen_direct_lighting]]（未着手）
  - SDF ヒットからのカード変換 → [[lumen_sw_rt]]
