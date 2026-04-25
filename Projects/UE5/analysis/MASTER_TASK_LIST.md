# UE5 解析 マスタータスクリスト

全システムの進捗を一元管理する。  
各フェーズ完了時にこのファイルのチェックボックスを更新すること。

- 更新日: 2026-04-20

---

## Rendering（ドキュメント完了済み）

- [x] CPU ドキュメント（23 フォルダ・416 本）
- [x] GPU ドキュメント（17 フォルダ）
- [x] ソースマップ システムレベル（`_source_map.md` 1 本、2026-04-19 完了）
- [x] ソースマップ Stage1: コアインフラ（5 本: SceneRenderer / RDG / RHI / MeshPassProcessor / GPUScene、2026-04-19 完了）
- [x] ソースマップ Stage2: 主要パス（8 本: DepthPrepass / BasePass / DeferredLighting / Shadow / VSM / Nanite / Lumen / PostProcess、2026-04-19 完了）
- [x] ソースマップ Stage3: 特化機能（10 本: HZB / Translucency / SkyAtmosphere / Fog / Decals / SSAO / DistanceField / RayTracing / Substrate / MegaLights、2026-04-19 完了）
- [x] ソースマップ Stage4: GPU 側（17 本: BasePass / Decals / DeferredLighting / DepthPrepass / Fog / GPUScene / HZB / MegaLights / PostProcess / RayTracing / SSAO / SkyAtmosphere / Translucency / Lumen / Nanite / VirtualShadowMaps / DistanceField、2026-04-19 完了）
- [x] GPU レンダーグラフ シリーズ（6 本: 03_overview + 04-08 Phase A〜E、2026-04-20 完了）  
  Mermaid `flowchart` でパスとリソースの依存関係を可視化。Modern / Legacy / AsyncCompute を図示。
- [ ] **Ph6: Algorithms 紐付け**（手順: `HOWTO_create_algorithm_docs.md`）
  - [ ] `Rendering/Algorithms/_algorithm_index.md` 作成
  - [ ] BRDF（GGX / Smith-Joint / Schlick / Disney Diffuse）
  - [ ] Lumen（Surface Cache / Radiance Cache / SW・HW Ray Tracing）
  - [ ] Nanite（Cluster Culling / Visibility Buffer / SW Rasterizer）
  - [ ] VSM（Virtual Shadow Maps、Karis 2021）
  - [ ] TAA / TSR（Karis 2014, Epic SIGGRAPH 2022）
  - [ ] Screen-Space 系（SSGI / SSR / SSAO）
  - [ ] Tone Mapping（ACES / Filmic / AgX）
  - [ ] Bloom（Convolution / Mip-pyramid）

---

## 非 Rendering 11 システム

各システムの本体ドキュメント（Ph0〜Ph4）完了後に **Ph5: Algorithms 紐付け** を実施する。  
手順: `HOWTO_create_algorithm_docs.md` 参照。
Algorithms フェーズはシステム本体完了とは独立に進めて良い（後回しで OK）。

### GAS（Gameplay Ability System）— 32 ファイル

- [ ] Ph0: ソースマップ（`_source_map.md`）
- [ ] Ph1: 概要（`01_gas_overview.md` 更新 + サブフォルダ概要 5 本）
- [ ] Ph2: Details（16 本）
- [ ] Ph3: Reference（9 本）
- [ ] Ph4: Flow 追記
- [ ] Ph5: Algorithms（クライアント予測ロールバック / Tag 集合演算）

### Animation — 34 ファイル

- [ ] Ph0: ソースマップ（`_source_map.md`）
- [ ] Ph1: 概要（`01_animation_overview.md` 更新 + サブフォルダ概要 6 本）
- [ ] Ph2: Details（17 本）
- [ ] Ph3: Reference（9 本）
- [ ] Ph4: Flow 追記
- [ ] Ph5: Algorithms（IK: CCD/FABRIK/TwoBone / Slerp / Motion Matching 距離関数）

### AI — 36 ファイル

- [ ] Ph0: ソースマップ（`_source_map.md`）
- [ ] Ph1: 概要（`01_ai_overview.md` 更新 + サブフォルダ概要 6 本）
- [ ] Ph2: Details（20 本）
- [ ] Ph3: Reference（8 本）
- [ ] Ph4: Flow 追記
- [ ] Ph5: Algorithms（A* / NavMesh / Steering Reynolds 1999 / EQS スコアリング）

### Core — 37 ファイル

- [x] Ph0: ソースマップ（`_source_map.md`、2026-04-24 完了）
- [x] Ph1: 概要（`01_core_overview.md` 更新 + サブフォルダ概要 6 本、2026-04-24 完了）
- [x] Ph2: Details（21 本、2026-04-24 完了）
- [x] Ph3: Reference（8 本、2026-04-25 完了）
- [x] Ph4: Flow 追記（28 本: overview 7 + Details 21、2026-04-25 完了）
- [ ] Ph5: Algorithms（FName ハッシュ / GC Mark&Sweep / TaskGraph スケジューリング）

### Physics — 35 ファイル

- [ ] Ph0: ソースマップ（`_source_map.md`）
- [ ] Ph1: 概要（`01_physics_overview.md` 更新 + サブフォルダ概要 6 本）
- [ ] Ph2: Details（19 本）
- [ ] Ph3: Reference（8 本）
- [ ] Ph4: Flow 追記
- [ ] Ph5: Algorithms（Chaos PBD/XPBD / Gauss-Seidel ソルバ / Cloth PBD / Pacejka Tire）

### Niagara — 34 ファイル

- [ ] Ph0: ソースマップ（`_source_map.md`）
- [ ] Ph1: 概要（`01_niagara_overview.md` 更新 + サブフォルダ概要 5 本）
- [ ] Ph2: Details（19 本）
- [ ] Ph3: Reference（8 本）
- [ ] Ph4: Flow 追記
- [ ] Ph5: Algorithms（Verlet/Symplectic / Curl Noise Bridson 2007 / GPU Sort Bitonic・Radix）

### Network — 34 ファイル

- [ ] Ph0: ソースマップ（`_source_map.md`）
- [ ] Ph1: 概要（`01_network_overview.md` 更新 + サブフォルダ概要 6 本）
- [ ] Ph2: Details（19 本）
- [ ] Ph3: Reference（7 本）
- [ ] Ph4: Flow 追記
- [ ] Ph5: Algorithms（Bandwidth/Priority / Snapshot 補間外挿 / Reliable Bunch 信頼転送）

### GameFramework — 31 ファイル

- [x] Ph0: ソースマップ（`_source_map.md`、2026-04-25 完了）
- [x] Ph1: 概要（`01_gameframework_overview.md` 更新 + サブフォルダ概要 5 本、2026-04-25 完了）
- [x] Ph2: Details（17 本、2026-04-25 完了）
- [x] Ph3: Reference（7 本、2026-04-25 完了）
- [x] Ph4: Flow 追記（23 本: overview 6 + Details 17、2026-04-25 完了）
- [ ] Ph5: Algorithms（CMC ネット予測 SavedMove リプレイ）

### WorldBuilding — 31 ファイル

- [x] Ph0: ソースマップ（`_source_map.md`）
- [x] Ph1: 概要（`01_worldbuilding_overview.md` 更新 + サブフォルダ概要 5 本）
- [x] Ph2: Details（17 本）
- [x] Ph3: Reference（7 本）
- [x] Ph4: Flow 追記（23 本: overview 1 + sub-overview 5 + Details 17）
- [ ] Ph5: Algorithms（World Partition Spatial Hash / HLOD Quadric Error / Streaming 距離・Frustum）

### Audio — 27 ファイル

- [ ] Ph0: ソースマップ（`_source_map.md`）
- [ ] Ph1: 概要（`01_audio_overview.md` 更新 + サブフォルダ概要 5 本）
- [ ] Ph2: Details（14 本）
- [ ] Ph3: Reference（6 本）
- [ ] Ph4: Flow 追記
- [ ] Ph5: Algorithms（HRTF 空間化 / Schroeder・FDN リバーブ / Biquad フィルタ）

### Input — 18 ファイル

- [ ] Ph0: ソースマップ（`_source_map.md`）
- [ ] Ph1: 概要（`01_input_overview.md` 更新 + サブフォルダ概要 3 本）
- [ ] Ph2: Details（9 本）
- [ ] Ph3: Reference（4 本）
- [ ] Ph4: Flow 追記
- [ ] Ph5: Algorithms（入力スタッキング優先度 / EnhancedInput Modifier・Trigger 数学）

---

## 全体サマリ

| カテゴリ | 完了 | 残り |
|---------|------|------|
| Rendering ドキュメント | 416 本 | 0 |
| Rendering ソースマップ | 41 | 0 |
| Rendering レンダーグラフ | 6 | 0 |
| 非 Rendering 11 システム | 0 | 339 |
| Algorithms（全 12 システム、Ph5/Ph6） | 0 | 12 システム分 |
| **合計** | **463** | **339+** |

---

## 推奨ワークフロー（1 システムあたり）

```
1. システムを1つ選ぶ
2. /model opus  → Ph0（ソースマップ）+ Ph1（概要）
3. /model sonnet → Ph2（Details）+ Ph3（Reference）をサブフォルダ単位で
4. /model opus  → Ph4（Flow 追記）
5. /model opus  → Ph5/Ph6（Algorithms 紐付け、`HOWTO_create_algorithm_docs.md` 参照）
6. MASTER_TASK_LIST.md の該当フェーズに [x] を付ける
```

Algorithms フェーズは独立して進めて良い（Ph0〜Ph4 完了後でなくても、後追いで OK）。  
WebSearch / WebFetch を使うため、事前に `ToolSearch` で `select:WebSearch,WebFetch` を読み込むこと。
