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
│   ├── _algorithm_index.md        ← 採用アルゴリズム一覧（理論⇄UE実装の対応表）
│   ├── _source_index.md           ← 取得済み外部資料インデックス + 未解決事項
│   ├── _papers/                   ← ローカルPDFキャッシュ（.gitignore で除外）
│   │   ├── _README.md             ← キャッシュ運用説明（コミット対象）
│   │   ├── karis_2013.pdf         ← 公開PDFのローカルコピー
│   │   └── walter_2007_summary.txt← 取得困難な論文の要旨抜粋
│   ├── {topic}.md                 ← 個別アルゴリズム解説（コミット対象）
│   ├── {topic}_full.md            ← 長文転載・詳細メモ（.gitignore で除外）
│   └── ...
└── TASK_CHECKLIST.md
```

### ナビゲーションファイルの役割分担

| ファイル | 役割 | コミット |
|--------|------|--------|
| `_algorithm_index.md` | 「アルゴリズム → 出典 → UE 実装ファイル」三方向対応表 | ◯ |
| `_source_index.md` | 取得済み資料一覧 + UE 実装インデックス + 未解決事項（相談トリガー） | ◯ |
| `_papers/_README.md` | キャッシュフォルダの説明・取得元メモ | ◯ |
| `_papers/*.pdf` | 公開論文 PDF のローカルコピー | × |
| `_papers/*_summary.txt` | 取得困難資料の要旨手書きメモ | × |
| `{topic}.md` | 個別アルゴリズム解説（自分の解説 + 短い引用） | ◯ |
| `{topic}_full.md` | 論文本文の長文転載 + 詳細個人メモ | × |

### 著作権ポリシー

- **コミット対象 (`{topic}.md`)**: 自分の解説文を主体とし、引用は数式 + 説明文の最小限に留める。出典明記必須。
- **長文転載 (`{topic}_full.md`)**: 個人学習用にローカルのみ。`.gitignore` で除外済み。論文本文の長い引用、画像コピー、Epic 公式講演スライドのテキスト化等を自由に置く。
- **PDF キャッシュ (`_papers/`)**: 再配布不可資料を含むため `.gitignore` で除外済み。Claude が次回 Read で参照可能。

---

## 2. ドキュメント テンプレート

### 2.0 `_source_index.md`（資料インデックス・相談用）

セッション中に Claude が「論文 ID から URL/ローカルパス/重要ページ」を即引きできる中央ハブ。

```markdown
# {SystemName} Algorithms ソース索引

## 取得済み外部資料

| ID | タイトル | 著者 | 年 | 種別 | URL | ローカル | 重要箇所 |
|----|--------|-----|----|----|-----|--------|--------|
| S01 | Real Shading in Unreal Engine 4 | Karis | 2013 | SIGGRAPH course | https://... | `_papers/karis_2013.pdf` | p.4-6 BRDF, p.10 Fresnel |
| S02 | Microfacet Models for Refraction | Walter | 2007 | 論文 | https://... | （PDF不可・要旨のみ） | Eq.33 GGX NDF |
| S03 | Understanding the Masking-Shadowing Function | Heitz | 2014 | JCGT | https://... | `_papers/heitz_2014.pdf` | Section 5 Smith G2 |

## UE 実装インデックス（このシステム内の数式関連ファイル）

| ファイル | 主要関数 | 採用アルゴリズム ID | 個別ドキュメント |
|--------|--------|-----------------|---------------|
| `BRDF.ush` | `D_GGX` | S01, S02 | [[brdf_ggx]] |
| `BRDF.ush` | `Vis_SmithJointApprox` | S03 | [[brdf_smith]] |
| `ShadingModels.ush` | `DiffuseBurley` | S04 | [[brdf_disney_diffuse]] |

## 未解決事項（相談トリガー）

| 項目 | 個別ドキュメント | 質問内容 | ステータス |
|------|---------------|--------|--------|
| `a2 = a*a` の意味 | [[brdf_ggx]] | なぜ Roughness を二乗するのか | 未解決 |
| Smith-Joint の Joint とは | [[brdf_smith]] | 元論文の Joint の定義 | 解決済（{topic}_full.md p.X 参照） |

## メンテナンス指針

- 新しい論文を取得したら `_papers/` に保存し本表に追記
- 「相談用フック」で挙がった疑問は「未解決事項」表に転記
- 解決した項目はステータスを更新（削除しない、後で振り返れるように）
```

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

- {論文タイトル} ({著者}, {年}) — {URL}（出典 ID: S01）
- SIGGRAPH course "Physically Based Shading in Theory and Practice" {年} — {URL}（S02）
- Epic 公式技術ブログ — {URL}

## 7. 相談用フック

セッション中に Claude が即提示できる「相談の起点」。長文転載や追加調査が必要なものは `{topic}_full.md` 参照。

- **理解度チェック**: なぜこの近似で十分なのか？ → `{topic}_full.md` 「§2 近似の妥当性検証」
- **次の派生**: Substrate ではどう変わるか？ → [[../../{OtherSystem}/Algorithms/...]]
- **未読箇所**: 出典 S02 の Section 5（Importance Sampling）— 後で読む
- **コード深掘り候補**: `BRDF.ush:120` の `D_GGX` 内部の `rcp` 最適化の妥当性
```

### 2.3 `{topic}_full.md`（長文転載・個人メモ、`.gitignore` 対象）

```markdown
# {アルゴリズム名} - 詳細メモ・長文転載

⚠️ このファイルは個人学習用ローカルキャッシュ。GitHub にプッシュされない。

- 対応する公開ドキュメント: [[{topic}]]
- 元資料: 出典 ID S01（`_papers/karis_2013.pdf`）

---

## §1 元論文の該当章節（転載）

> （論文の該当ページから引用した本文・図表の説明・数式の導出過程）

## §2 近似の妥当性検証（個人メモ）

（数式を手で展開した結果、検証スクリプト実行結果、誤差プロット等）

## §3 Claude との相談ログ

（過去の質疑応答で重要だったやりとりを保存）
```

### 2.4 `_papers/_README.md`（キャッシュ運用説明、コミット対象）

```markdown
# Papers キャッシュ

このフォルダは外部資料（論文 PDF、SIGGRAPH course notes 等）のローカルキャッシュ。  
**`.gitignore` で除外されているため、リポジトリには含まれない。**

## 取得方法

各資料の取得元 URL は `../_source_index.md` を参照。  
公開 PDF はダウンロードしてここに置く。ファイル名規則: `{著者姓}_{年}_{短いタイトル}.pdf`

## 取得困難な資料

IEEE / ACM 有料論文等は PDF を置かず、要旨を `{著者姓}_{年}_summary.txt` として手書きで残す。
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

### Step 3: `_algorithm_index.md` + `_source_index.md` 作成

抽出したアルゴリズム一覧と取得済み資料一覧を表形式でまとめる（個別ドキュメント未作成でも先に索引 2 本を完成させる）。  
公開 PDF はこの段階で `_papers/` にダウンロードしておく。

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
- [ ] `_source_index.md` に取得済み資料と未解決事項が記載されている
- [ ] `_papers/_README.md` が存在する（PDF 自体は `.gitignore` 対象）
- [ ] 各アルゴリズムにつき 1 ドキュメント（or 1 索引行 + 後回し明記）
- [ ] 既存 Details から Algorithms への相互リンクが張られている
- [ ] `MASTER_TASK_LIST.md` の該当 Algorithms 行が `[x]`

---

## 8. 相談セッションの効率化

ユーザーから「{topic} の {部分} について相談したい」と言われたら、以下の順で読む:

1. `Algorithms/_source_index.md` — 出典 ID と未解決事項を即把握
2. `Algorithms/{topic}.md` — 公開ドキュメント本体
3. `Algorithms/{topic}_full.md`（あれば） — 長文転載・詳細メモ
4. `Algorithms/_papers/{paper}.pdf`（あれば） — 元資料の該当ページのみ Read

相談中に新たな疑問が出たら `_source_index.md` の「未解決事項」表に追記する。  
セッション後に解決したらステータス更新。
