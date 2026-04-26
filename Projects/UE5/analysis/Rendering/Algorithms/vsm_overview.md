---
name: Virtual Shadow Maps Overview
description: Sparse Pages による 16K×16K 仮想シャドウマップの理論と UE 実装（Karis 2021）
type: project
---

# Virtual Shadow Maps（VSM）— Sparse Pages による超高解像度シャドウ

- 上位: [[_algorithm_index]]
- 関連: [[vsm_page_streaming]] / [[shadow_pcf_pcss]] / [[nanite_cluster_culling]]
- 採用システム: UE5 標準シャドウ（Directional / Local Light）
- 出典 ID: **S21**（[[_source_index]]）— Karis 2021 SIGGRAPH / Epic 公開ドキュメント
- 影響元: Sparse Virtual Texturing (SVT, id Tech 5), Mike Day "Cascaded Shadow Maps"

---

## 1. 何のためのアルゴリズムか

従来 CSM (Cascaded Shadow Maps) では:
- カスケード切替で **解像度の段差**（ピクセル境界で aliasing）
- 全カスケードが固定解像度マップ（例 4K×4K × 4 段）= **常時 64MB+**
- 局所ライト（点光源）ごとに別マップが必要 → スケールせず

### 素朴な手法の問題

- **巨大なシャドウマップ（16K²）を全部割り当て**: 1GB+ で破綻
- **CSM 多段化**: カスケード境界の段差解消できない
- **Adaptive Shadow Maps**: 旧来研究は CPU 律速で実用化困難

### VSM の貢献

- **仮想 16K×16K シャドウマップ**を全光源に与える（仮想アドレス空間）
- 物理メモリは **128×128 物理ページプール**から動的割当（典型 2048 ページ = 32MB）
- **画面のシャドウレシーバが必要とするページだけ**マーク → 割当 → ラスタ
- Nanite と統合: ページ単位カリング、ページ単位ラスタ
- フレーム間 **ページキャッシュ** で動かない領域を再利用

---

## 2. 理論

### 2.1 仮想 / 物理ページの分離

| 概念 | サイズ | 役割 |
|------|------|------|
| **仮想ページ** | 128×128 = 16384 px × 16384 px の仮想アドレス空間 | 各ライトの「論理的な」シャドウマップ |
| **物理ページ** | 128×128 px、典型 2048 個 | GPU メモリ上の実体（共有プール） |
| **ページテーブル** | 仮想 → 物理マッピング | 各レシーバピクセルから物理ページを引く |

`VirtualShadowMapDefinitions.h:13-20`:
```c
#define VSM_LOG2_PAGE_SIZE 7u                                             // 128
#define VSM_PAGE_SIZE (1u << VSM_LOG2_PAGE_SIZE)
#define VSM_LOG2_LEVEL0_DIM_PAGES_XY 7u                                   // 128 ページ × 128 ページ
#define VSM_LEVEL0_DIM_PAGES_XY (1u << VSM_LOG2_LEVEL0_DIM_PAGES_XY)
#define VSM_MAX_MIP_LEVELS (VSM_LOG2_LEVEL0_DIM_PAGES_XY + 1u)            // 8 ミップ
#define VSM_VIRTUAL_MAX_RESOLUTION_XY (VSM_LEVEL0_DIM_PAGES_XY * VSM_PAGE_SIZE)  // 16384
```

### 2.2 Mip 階層（仮想ミップマップ）

仮想 16384² → 1 段下げると 8192² → ... → 128²（ミップ 8 段）。レシーバーピクセルから **シャドウサンプリング距離に応じた mip level** を選び、適切な解像度ページを参照。

CSM は明示的なカスケード境界が見えるが、VSM は mip 階層が連続なので段差が消える。

### 2.3 ページマーキング（必要ページの抽出）

毎フレーム G-Buffer の各ピクセルから:
1. シャドウマップ上のサンプル座標を計算
2. 対応する **仮想ページ ID** をマーク（`MarkPagesUsedByPixel`）
3. 重複除去 → 必要ページリスト構築

`r.Shadow.Virtual.MarkPagesUsingFroxels = 1` で 8x8 froxel 単位の近似マーキング（高速）。

### 2.4 ページ割当 / ラスタ

1. 必要ページリスト ↔ 物理プールのマッピングテーブルを更新
2. キャッシュ済みページはスキップ
3. 未割当ページに物理スロットを割当（フリーリスト）
4. 各物理ページに対し Nanite ラスタを起動 → クリア → ジオメトリラスタ

ページテーブルは `(LightIndex, MipLevel, X, Y) → PhysicalPageIndex` 構造。

### 2.5 Nanite との統合

Nanite クラスタカリング（[[nanite_cluster_culling]]）は VSM ページを「複数ビュー」として扱う:
- 1 物理ページ = 1 ミニビュー（128×128 のミニラスタ）
- カリング passは `VIRTUAL_TEXTURE_TARGET=1` ブランチでページ単位に振り分け
- `r.Nanite.VSMInvalidateOnLODDelta` で LOD 不一致時のキャッシュ無効化

非 Nanite メッシュは **per-page draw command 構築** → ページ単位 instance ラスタ。

### 2.6 Directional Light 用 Clipmap

ディレクショナルライトはカメラに追従し巨大範囲をカバーする必要があるので、**Clipmap 階層**:

| Clipmap Level | 範囲 | 解像度 |
|--------------|------|------|
| 0 | カメラ近傍（小） | 高 |
| ... | ... | ... |
| N | 遠方（大） | 低 |

各 Clipmap が独立した仮想 16K² を持ち、カメラ移動でロール（toroidal addressing）。`VirtualShadowMapClipmap.cpp` 参照。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `VirtualShadowMapArray.cpp` | VSM 全体オーケストレーション、ページプール |
| `VirtualShadowMapCacheManager.cpp` | フレーム間キャッシュ → [[vsm_page_streaming]] |
| `VirtualShadowMapClipmap.cpp` | ディレクショナルライト用 Clipmap |
| `VirtualShadowMapProjection.cpp` | レシーバ側サンプリング |
| `VirtualShadowMapDefinitions.h` | サイズ定数（VSM_PAGE_SIZE 等） |
| `VirtualShadowMapPageManagement.usf` | ページ割当 compute |
| `VirtualShadowMapPageMarking.usf` | レシーバから必要ページ抽出 |
| `VirtualShadowMapBuildPerPageDrawCommands.usf` | 非 Nanite per-page draw |

### 3.2 主要 CVar（基本）

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.Shadow.Virtual.Enable` | 0 (プロジェクト設定で 1) | VSM 有効化 |
| `r.Shadow.Virtual.MaxPhysicalPages` | 2048 | 物理ページプール容量 |
| `r.Shadow.Virtual.DynamicHZB` | 0 | 動的キャッシュ用に独立 HZB を構築 |
| `r.Shadow.Virtual.MarkPagesUsingFroxels` | 0 | 8x8 froxel 近似マーキング |
| `r.Shadow.Virtual.ShowLightDrawEvents` | 0 | ライト単位イベント表示 |
| `r.Shadow.Virtual.Stats.Visible` | false | 画面に統計表示 |

### 3.3 動的解像度（メモリ圧縮時）

ページプールが逼迫したら **解像度 LOD バイアス**で粗化:

```cpp
// VirtualShadowMapCacheManager.cpp:98-114
TAutoConsoleVariable<float> CVarVSMDynamicResolutionMaxLodBias(
    TEXT("r.Shadow.Virtual.DynamicRes.MaxResolutionLodBias"),
    2.0f,
    TEXT("As memory or compute-time cost limits are approached, VSM resolution ramps down...\n")
    TEXT("This is the maximum LOD bias to clamp to for global dynamic shadow resolution reduction. 0 = disabled"));
```

### 3.4 主要 CVar（解像度バイアス）

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.Shadow.Virtual.DynamicRes.MaxResolutionLodBias` | 2.0 | プール逼迫時の最大粗化バイアス |
| `r.Shadow.Virtual.DynamicRes.MaxComputeResolutionLodBiasDirectional` | 99999 | ディレクショナル専用 |
| `r.Shadow.Virtual.DynamicRes.MaxComputeResolutionLodBiasLocal` | 99999 | ローカルライト専用 |
| `r.Shadow.Virtual.AllocatePagePoolAsReservedResource` | 1 | プールを Reserved Resource で確保 |

### 3.5 Uniform Buffer

```cpp
IMPLEMENT_STATIC_UNIFORM_BUFFER_SLOT(VirtualShadowMapUbSlot);
IMPLEMENT_STATIC_UNIFORM_BUFFER_STRUCT(FVirtualShadowMapUniformParameters, "VirtualShadowMap", VirtualShadowMapUbSlot);
```

`VirtualShadowMap` UBO がレシーバ側シェーダ（lighting pass 等）にバインド。

---

## 4. 近似・省略の差分

| 項目 | 理想（無限メモリ） | UE VSM | 影響 |
|------|------------------|-------|------|
| 解像度 | 完全（数兆 px） | 仮想 16K² × ライト数 | 無視できる |
| 物理メモリ | 全ページ常駐 | プール 2048 ページ | プール逼迫で動的粗化 |
| Mip 連続性 | 完全 | 8 段 mip | 通常気にならない |
| ページ更新 | 即時 | キャッシュ + dirty 更新 | ライト/シャドウ変化で 1 フレーム遅延 |
| 非 Nanite ジオメトリ | 同等 | per-page draw command で代替 | drawcall コスト |

---

## 5. パラメータと CVar

§3.2, §3.4 にまとめ済み。

---

## 6. 代替手法との比較

| 手法 | 解像度設計 | 動的更新 | UE 採用 |
|------|----------|--------|--------|
| Static Shadow Map | 固定 N×N | 静的 | UE Stationary Light |
| **CSM** (Cascaded SM) | カスケード N 段 | 動的 | UE4 標準 / UE5 後方互換 |
| Sample Distribution SM | 視錐連動 | 動的 | DirectX SDK 系 |
| **VSM（Karis 2021）** | **仮想 16K² + Sparse Pages** | **ページ単位 + キャッシュ** | **UE5 標準** |
| Adaptive SM (Fernando 2001) | 解像度動的 | 動的 | 学術原典 |

### CSM vs VSM

- CSM: カスケード境界で aliasing、ライト数で線形増加
- VSM: mip 連続、ページ予算が共有プール → ライト数増えても破綻しにくい

---

## 7. 参考資料

- [S21] Karis 2021 SIGGRAPH "Virtual Shadow Maps"
- Epic 公式ドキュメント "Virtual Shadow Maps in Unreal Engine 5"
- id Tech 5 Sparse Virtual Texturing（メモリ仮想化の系譜）
- 関連: [[vsm_page_streaming]] / [[shadow_pcf_pcss]]

---

## 8. 相談用フック

- **理解度チェック**:
  - 仮想 / 物理ページの分離理由 → メモリ予算を共有プールにまとめる
  - mip 階層が CSM カスケードと違う点 → 連続的、境界 aliasing なし
  - Nanite と統合した利点 → ページ単位カリング/ラスタが直接適用可
- **コード深掘り候補**:
  - `VirtualShadowMapPageMarking.usf` のレシーバから必要ページ抽出
  - `VirtualShadowMapPageManagement.usf` のフリーリスト割当
  - `VirtualShadowMapBuildPerPageDrawCommands.usf` の非 Nanite drawcall 生成
- **未読箇所**:
  - S21 §3 LOD 選択式（仮想 mip と receiver pixel の対応）
  - `VirtualShadowMapClipmap.cpp` の toroidal アドレッシング
  - HW PCF / SMRT サンプリング統合（`VirtualShadowMapProjection.cpp`）
- **次の派生**:
  - キャッシュ / ストリーミング詳細 → [[vsm_page_streaming]]
  - PCF / PCSS フィルタ → [[shadow_pcf_pcss]]
  - SMRT (Stochastic Many-Light Ray Tracing) → 未着手
