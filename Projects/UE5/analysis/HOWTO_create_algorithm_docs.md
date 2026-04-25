# UE5 アルゴリズム紐付けドキュメント作成手順

各機能で採用された **アルゴリズム / 数式 / PBR モデル / 近似手法** を、UE5 の実装コードと外部論文/記事に紐付けて解説するドキュメント群を作成する。

`HOWTO_create_system_docs.md`（Ph0〜Ph5）の **追加フェーズ Ph6 = Algorithms** という位置付け。  
既存システム文書が完了した後に、その上に「理論レイヤ」を被せていく。

---

## 0. 目的

| やりたいこと | 既存文書（Ph0〜Ph5） | Algorithms 文書（本手順） |
|------------|------------------|--------------------------|
| クラス構造・呼び出しフロー | ◯ | △（要点のみ） |
| なぜその式・手法を選んだか | ×（コードからは不明） | ◯ |
| 論文・SIGGRAPH 講演との対応 | × | ◯ |
| 近似や省略の差分（理論式 vs UE 実装） | × | ◯ |
| 代替手法との比較 | × | ◯ |

---

## 1. フォルダ構成

各システムの解析フォルダ直下に `Algorithms/` を作る:

```
analysis/{SystemName}/
├── _source_map.md
├── 01_{system}_overview.md
├── {SubFolder}/...
├── Algorithms/                    ← 新規
│   ├── _algorithm_index.md        ← 採用アルゴリズム一覧 + 出典リンク
│   ├── {topic}.md                 ← 個別アルゴリズム解説
│   └── ...
└── TASK_CHECKLIST.md
```

`_algorithm_index.md` は **そのシステム内で採用されている全アルゴリズムの一覧表**。各行から個別ドキュメントへリンクする。

---

## 2. ドキュメント テンプレート

### 2.1 `_algorithm_index.md`（インデックス）

```markdown
# {SystemName} アルゴリズム索引

| 機能領域 | アルゴリズム | 出典（年） | 個別ドキュメント | UE 実装ファイル |
|---------|-----------|----------|---------------|--------------|
| BRDF | GGX (Trowbridge-Reitz) | Walter 2007 | [[brdf_ggx]] | `BRDF.ush` |
| BRDF | Disney Diffuse | Burley 2012 | [[brdf_disney_diffuse]] | `ShadingModels.ush` |
| GI   | Radiance Cache + Surface Cache | Lumen SIGGRAPH 2022 | [[lumen_surface_cache]] | `Lumen*.cpp/.usf` |
```

### 2.2 個別アルゴリズム ドキュメント `{topic}.md`

```markdown
# {アルゴリズム名}（日本語タイトル）

- 上位: [[_algorithm_index]]
- 関連既存ドキュメント: [[../{SubFolder}/Details/x_yyy]]
- 採用システム: {SubsystemName}
- 出典: {論文タイトル}, {著者} ({年})
- 出典 URL: {論文 PDF or SIGGRAPH course notes URL}

---

## 1. 何のためのアルゴリズムか

{1〜2 段落の問題設定: 何を計算したいのか、素朴な手法だと何が困るのか}

## 2. 理論

### 数式（理論式）

```math
f_r(l, v) = \frac{D(h) F(v, h) G(l, v, h)}{4 (n \cdot l)(n \cdot v)}
```

各項の意味:
- `D(h)`: 法線分布関数 (NDF)
- `F(v, h)`: フレネル項
- `G(l, v, h)`: 幾何遮蔽項

### 導出の要点

{論文での導出ステップを 3〜5 ポイントで}

## 3. UE での実装

### 採用された具体形

UE は `D` に GGX、`G` に Smith-Joint、`F` に Schlick 近似を使用。

### コードとの対応

| 理論項 | UE 実装関数 | ファイル:行 |
|------|----------|----------|
| `D(h)` GGX | `D_GGX(a2, NoH)` | `BRDF.ush:120` |
| `G(l,v,h)` Smith | `Vis_SmithJointApprox()` | `BRDF.ush:240` |
| `F(v,h)` Schlick | `F_Schlick()` | `BRDF.ush:300` |

### 近似・省略の差分

| 項目 | 理論 | UE 実装 | 影響 |
|------|------|--------|------|
| Roughness 変換 | `α = roughness²` | 同左 | なし |
| Smith G | 完全形（Lambda 関数） | Joint 近似 | 高 Roughness で僅差 |

## 4. パラメータと CVar

| CVar / パラメータ | デフォルト | 範囲 | 効果 |
|-----------------|----------|------|------|
| `r.Material.RoughnessOverride` | -1 | 0-1 | デバッグ用 |

## 5. 代替手法との比較

| 手法 | 物理的妥当性 | コスト | UE での採用 |
|------|-----------|------|----------|
| Phong | × | 低 | レガシー |
| Blinn-Phong | △ | 低 | レガシー |
| Cook-Torrance + GGX | ◯ | 中 | **採用** |
| Layered BSDF | ◯+ | 高 | Substrate のみ |

## 6. 参考資料

- {論文タイトル} ({著者}, {年}) — {URL}
- SIGGRAPH course "Physically Based Shading in Theory and Practice" {年} — {URL}
- Epic 公式技術ブログ — {URL}
```

---

## 3. ワークフロー

### Step 1: 索引候補の抽出（ソースから）

対象システムのシェーダ (`.usf`/`.ush`) と CPP を Grep で走査:

```
論文/著者参照のキーワードを Grep:
  - "Karis", "Heitz", "Burley", "Hammon", "Walter", "Schlick", "GGX"
  - "based on", "from paper", "see paper", "reference:"
  - 4桁の年号（コメント中の "2013" 等）
```

UE の Rendering 系シェーダは特にコメントで論文参照が豊富。GAS / Animation / Physics でも数学的処理（IK の CCD/FABRIK、剛体ソルバ、予測補正等）に同様の参照がある。

### Step 2: WebSearch / WebFetch で出典確定

1. 抽出キーワードで `WebSearch` → 論文 PDF / SIGGRAPH course notes URL を取得
2. `WebFetch` で要旨・主要数式・アルゴリズム説明を取得
3. 信頼できる出典のみ採用（Epic 公式 / SIGGRAPH / ACM / 著名研究者個人サイト）

### Step 3: `_algorithm_index.md` 作成

抽出したアルゴリズム一覧を表形式でまとめる（個別ドキュメント未作成でも先に索引だけ完成させる）。

### Step 4: 個別ドキュメント作成（優先度順）

採用箇所が広い・コア機能のものから順に作成:

```
優先度 高: BRDF（全マテリアルで使用）, GI 基幹（Lumen 等）
優先度 中: 影（VSM）, ポストエフェクト（TAA, Bloom, DOF）
優先度 低: ニッチ機能（特殊エフェクト等）
```

### Step 5: 既存 Details からの双方向リンク

既存 Details ドキュメント (`Details/x_yyy.md`) の「関連」セクションに `[[../Algorithms/{topic}]]` を追記し、相互参照する。

---

## 4. システム別の主要ターゲット例

### Rendering
- BRDF: Disney Principled, GGX, Smith-Joint, Schlick
- TAA / TSR: Karis 2014, Epic SIGGRAPH 2022
- Lumen: Software/Hardware Ray Tracing, Surface Cache, Radiance Cache
- Nanite: Cluster culling, Visibility Buffer, Software rasterizer
- VSM: Karis 2021 (Virtual Shadow Maps)
- SSGI / SSR / SSAO の screen-space 近似
- Tone mapping: ACES / Filmic / AgX
- Bloom: Convolution / Mip-pyramid

### Animation
- IK: CCD, FABRIK, Two-Bone Analytical
- Pose Blending: Slerp / NLerp の数学
- AnimationBudgetAllocator のスケジューリング
- Motion Matching の特徴量距離

### Physics
- Chaos Solver: PBD / XPBD
- Constraint solver: Gauss-Seidel / Jacobi
- Cloth: Position-Based Dynamics
- Vehicle: PacejkaTire model

### AI
- Path: A* / NavMesh detail mesh
- Steering: Reynolds 1999
- EQS: spatial query scoring
- Perception: stimulus aging

### Niagara
- Particle integration: Verlet / Symplectic
- Curl Noise (Bridson 2007)
- GPU Sort: Bitonic / Radix

### Audio
- Spatialization: HRTF, ITD/ILD
- Reverb: Schroeder / Feedback Delay Network
- DSP filters: Biquad

### GAS
- Client prediction & rollback
- Tag-based gating の集合演算

### Network
- Replication: Bandwidth allocation, Priority
- Snapshot interpolation / extrapolation
- Reliable bunches の信頼転送

### Core
- FName ハッシュ / FString → FNameEntry 変換
- Garbage Collection: Mark & Sweep + reachability analysis
- Async task graph スケジューリング

### GameFramework
- CMC ネット予測の SavedMove リプレイ

### WorldBuilding
- World Partition: Spatial Hash Grid
- HLOD: Mesh simplification (Quadric Error)
- Streaming: Distance-based + Frustum culling

### Input
- 入力スタッキング・優先度
- EnhancedInput Modifier / Trigger の数学（Dead Zone 等）

---

## 5. モデル選択

| フェーズ | 推奨モデル | 理由 |
|---------|----------|------|
| 索引候補抽出（Grep + 集計） | Sonnet | 機械的処理 |
| 出典確定（WebSearch + WebFetch） | Sonnet/Opus | 検索結果の判定 |
| 個別ドキュメント執筆 | **Opus** | 数式・近似差分の正確性が重要 |
| 既存 Details への相互リンク追記 | Sonnet | 機械的編集 |

---

## 6. WebSearch / WebFetch 利用ノート

- **遅延ツール**: 使用前に `ToolSearch` で `select:WebSearch,WebFetch` を読み込む必要がある
- **取得困難な出典**: IEEE / ACM Digital Library の有料論文は本文取得不可 → アブストラクト + Epic 解説資料で代替
- **信頼度マーキング**: 出典に応じて `[公式]` `[論文]` `[個人ブログ]` を明記

---

## 7. 完了条件

各システムの Algorithms フェーズ完了は以下を満たすこと:
- [ ] `_algorithm_index.md` がそのシステムの主要アルゴリズムを網羅している
- [ ] 各アルゴリズムにつき 1 ドキュメント（or 1 索引行 + 後回し明記）
- [ ] 既存 Details から Algorithms への相互リンクが張られている
- [ ] `MASTER_TASK_LIST.md` の該当 Algorithms 行が `[x]`
