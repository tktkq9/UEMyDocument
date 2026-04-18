# UE5 サブシステム 解析ドキュメント作成手順

Lumen（40本完了）で確立した解析ドキュメント作成方式を汎用化。  
Rendering 以外のシステム（GAS, Animation, AI 等）にも同じ構成で使える。

---

## 0. 用語

| 変数 | 意味 | 例 |
|------|------|-----|
| `{SystemName}` | 解析対象システム名 | `GAS`, `Animation`, `Lumen` |
| `{SourceRoot}` | UE5 ソースのルートパス | `D:\UnrealEngine\` |
| `{AnalysisRoot}` | 解析メモのルートパス | `D:\Learning\Projects\UE5\analysis\` |
| `{ModulePath}` | 対象モジュールのソースパス（`_module_index.md` で確認） | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/` |

---

## 1. フォルダ構成

### 1.1 階層の考え方

フォルダ階層は **ソースの複雑さに応じて** 決定する。一律に同じ深さにする必要はない。

| 階層パターン | いつ使うか | 例 |
|------------|----------|-----|
| **2階層**: `System/SubFolder/Details/` | 標準。ほとんどのシステムはこれで十分 | `GAS/AbilitySystem/Details/` |
| **3階層**: `System/SubFolder/Reference/GroupName/` | Reference が **10本以上** になった場合 | `Rendering/Lumen/Reference/a_SurfaceCache/` |
| **フラット**: `System/Details/` | サブフォルダが不要な小規模システム（ソースファイル 20 本以下） | `Input/EnhancedInput/Details/`（現状これで十分） |

**判断基準（フェーズ 0 のソースマップ作成時に決定）:**

1. **サブフォルダ化**: 対象ソースが **複数の独立した機能領域** を持ち、各領域に 3 本以上の Details が見込まれる → サブフォルダ化する
2. **Reference グループ化**: 1 サブフォルダ内の Reference が **10本を超える** → `Reference/` 内をグループフォルダで分割する
3. **深掘り判断**: 作業中に Details 1 本がカバーする範囲が広すぎると感じたら → Details を分割して増やす（フォルダ階層を増やすのではなく）

**やってはいけないこと:**
- 最初から3階層以上を作り込む（ファイル数が増えてから対応で十分）
- 1ファイルしか入らないフォルダを作る（その場合は上位フォルダに統合する）

### 1.2 標準フォルダ構成（完成形）

```
{AnalysisRoot}/{SystemName}/
├── _source_map.md              ← ソースナビゲーション用（フェーズ 0）
├── 01_{system}_overview.md     ← システム全体概要
├── {SubFolder}/                ← 機能領域ごとのサブフォルダ
│   ├── 01_overview.md          ← サブフォルダ概要
│   ├── Details/
│   │   ├── a_{topic}.md        ← サブシステム別の概念記事
│   │   └── ...
│   ├── Reference/
│   │   ├── ref_{name}.md       ← ソースファイル別API参照
│   │   └── ...（10本超なら下記のグループ化構成へ）
│   └── TASK_CHECKLIST.md       ← サブフォルダ単位の進捗管理（必要に応じて）
├── Dives/                      ← アドホック調査メモ（ue5-dive で生成）
│   └── dive_{topic}.md
└── TASK_CHECKLIST.md           ← システム全体の進捗管理
```

### 1.3 Reference グループ化構成（大規模時）

Reference が 10 本を超えた場合、Details のグループに対応させてフォルダ分割する:

```
{SubFolder}/Reference/
├── Common/                     ← 共通クラス・ヘッダー
│   └── ref_{system}_core.md
├── a_{GroupName}/              ← Details/a_*.md に対応
│   ├── ref_{file1}.md
│   └── ref_{file2}.md
└── b_{GroupName}/
    └── ref_{file3}.md
```

### 1.4 Lumen での実例（3階層 Reference）

```
analysis/Rendering/Lumen/
├── 02_lumen_overview.md
├── Details/
│   ├── a_lumen_surface_cache.md
│   └── ...（6本）
├── Reference/
│   ├── Common/                 ← Reference 34本 → グループ化
│   ├── a_SurfaceCache/
│   ├── b_SceneLighting/
│   └── ...
└── TASK_CHECKLIST.md
```

---

## 2. 作成フェーズ

### フェーズ 0：ソースマップ作成 + 階層判断

**目的**: Claude が効率的にソースを探索できるナビゲーション情報を整備し、フォルダ階層を決定する。

1. `_module_index.md` で対象システムのソースパスを確認
2. `{SourceRoot}/{ModulePath}` を Glob でスキャンし、ファイル一覧を取得
3. `Templates/ue5_source_map.md` をベースに `_source_map.md` を作成
4. 主要ファイル→クラス対応表、エントリポイント表を埋める
5. **階層判断**: ソースファイル数と機能領域の独立性に基づき、TASK_CHECKLIST の構成を確認・調整する

**階層判断のフローチャート:**
```
ソースファイルを機能領域でグループ化
  │
  ├─ グループが 1〜2 個 → フラット構成（Details/ 直下）
  ├─ グループが 3 個以上 → サブフォルダ構成（既に TASK_CHECKLIST に反映済みならそのまま）
  │
  └─ 1グループ内の Reference 見込みが 10本超 → Reference/ 内をグループ化（フェーズ 3 で実施）
```

### フェーズ 1：概要 + チェックリスト

1. ソースフォルダのファイル一覧を取得し、サブシステムごとにグループ化
2. `TASK_CHECKLIST.md` 作成
3. `01_{system}_overview.md` 作成

**グループ化の基準:**
- 同じ機能領域（例: 剛体 / コリジョン / 破壊 / ビークル）
- 1グループ = 5〜8ファイル程度
- ヘッダ（.h）と実装（.cpp）は同じグループに

### フェーズ 2：Details 記事（グループ単位）

各サブシステムの概念記事。1グループ = 1ファイル。

```markdown
# {SystemName} {SubsystemName}（日本語タイトル）

- 上位: [[01_{system}_overview]]
- 関連: [[隣接する Details へのリンク]]

## 概要
## 全体フロー（Mermaid フローチャート）
## 主要クラス・構造体（コードブロック + 日本語コメント）
## 主要 CVar
## 関連ソースファイル
## 関連リファレンス    ← フェーズ 3 完了後に追加
```

### フェーズ 3：Reference 記事（グループ単位）

1ソースファイル（またはh+cppセット）= 1ファイル。

```markdown
# リファレンス：{FileName}.h / {FileName}.cpp

- グループ: {英字} - {SubsystemName}
- 上位: [[{対応する Details}]]
- ソース: `{ModulePath}/{FileName}.h/cpp`

## 概要
## 主要クラス・名前空間
## 主要関数
| 関数 | 引数 | 戻り値 | 説明 |

## 主要 CVar
| CVar | デフォルト | 説明 |
```

### フェーズ 4：コード実行フロー追記

概要・Details 各ファイルに `## コード実行フロー` セクションを追記する。

```markdown
## コード実行フロー

### エントリポイント
（ASCII コールツリー）

### フロー詳細
1. **{ステップ名}** — 説明（`File.cpp:行番号`）
   - 条件: `{CVar or 条件式}`
   - 参照: [[ref_xxx]]

### 関与クラス・関数一覧
| クラス / 関数 | ファイル | 役割 |
```

### フェーズ 5：Reference 拡張（オプション）

初期 Reference にメンバ変数表、パラメータ表、内部処理フロー、使用箇所リンクを追記。

---

## 3. ファイル命名規則

| 種類 | 命名パターン | 例 |
|------|------------|-----|
| ソースマップ | `_source_map.md` | `_source_map.md` |
| 概要 | `01_{system}_overview.md` | `01_gas_overview.md` |
| Details | `{alpha}_{topic}.md` | `a_ability_system.md` |
| Reference | `ref_{name}.md` | `ref_ability_system_component.md` |
| Dive | `dive_{topic}.md` | `dive_ge_application_flow.md` |
| チェックリスト | `TASK_CHECKLIST.md` | `TASK_CHECKLIST.md` |

---

## 4. Obsidian WikiLink のルール

```
概要 → Details:       [[a_ability_system]]
Details → Details:    - 関連: [[b_gameplay_effect]] | [[c_gameplay_cue]]
Reference → Details:  - 上位: [[a_ability_system]]
Details → Reference:  [[ref_ability_system_component]]
Dive → 関連:          [[01_gas_overview]] | [[a_ability_system]]
```

---

## 5. セッション運用

- **1セッション = 1グループ**（5〜8本）を目安
- グループ完了後に「次のグループに進みますか？」と確認
- `TASK_CHECKLIST.md` を常に最新に保つ
- セッション開始時は `_source_map.md` → `TASK_CHECKLIST.md` の順に Read

### 作業中の粒度調整

ドキュメント作成中に以下の状況が発生したら、構成を調整する:

| 状況 | 対応 |
|------|------|
| Details 1 本が長くなりすぎる（目安: 300行超） | Details を分割する。TASK_CHECKLIST に新ファイルを追加 |
| Reference が 10 本を超えた | `Reference/` 内にグループフォルダを作成する |
| サブフォルダ内の Details が 1 本しかない | 上位フォルダに統合を検討する |
| ソースマップ作成時に TASK_CHECKLIST と実態が乖離 | TASK_CHECKLIST を修正してからドキュメント作成に進む |
| 調査中に想定外のサブシステムを発見 | TASK_CHECKLIST にサブフォルダを追加し、ユーザーに報告する |

**原則**: 最初から完璧な構成を作り込むのではなく、作業中に育てる。ただし TASK_CHECKLIST は常に現状を正確に反映すること。

---

## 6. Skill の使い分け

| Skill | 用途 | いつ使う |
|-------|------|---------|
| `ue5-doc` | バッチ文書作成（本手順のフェーズ 0〜5） | 新システムの文書を体系的に作成するとき |
| `ue5-dive` | アドホック調査 | 「この関数の中身教えて」等の都度質問 |

---

## 7. Claude への依頼テンプレート

### 新システムのドキュメント作成

```
{SystemName}のソースコード解析ドキュメントを作成してください。

保存先: D:\Learning\Projects\UE5\analysis\{SystemName}\

手順:
1. _module_index.md でソースパスを確認
2. ソースマップ（_source_map.md）を作成
3. TASK_CHECKLIST.md を作成
4. 概要ファイルを作成
5. グループごとに確認を取りながら進める

参考:
- 手順: D:\Learning\Projects\UE5\analysis\HOWTO_create_system_docs.md
- Lumen 実例: D:\Learning\Projects\UE5\analysis\Rendering\Lumen\
```

### アドホック調査

```
{SystemName}の{クラス名/関数名}の実装を調べてください。
（→ ue5-dive Skill が自動発動）
```
