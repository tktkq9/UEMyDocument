# UE5 レンダリングシステム 解析ドキュメント作成手順

Lumen での実績をもとに、Nanite・VSM・その他レンダリングサブシステムで  
同じ構成のドキュメントを作るための手順書。

---

## 1. フォルダ構成（完成形）

```
analysis/Rendering/{SystemName}/
├── XX_{system}_overview.md          ← システム全体概要
├── Details/
│   ├── a_{system}_{subsystem}.md    ← サブシステム別の概念記事
│   ├── b_{system}_{subsystem}.md
│   └── ...
├── Reference/
│   ├── Common/                      ← システム共通クラス
│   ├── a_{SubsystemName}/           ← Detailsに対応したグループフォルダ
│   │   ├── ref_{system}_{file}.md
│   │   └── ...
│   └── ...
└── TASK_CHECKLIST.md                ← 進捗管理チェックリスト
```

### Lumen での実例

```
analysis/Rendering/Lumen/
├── 02_lumen_overview.md
├── Details/
│   ├── a_lumen_surface_cache.md
│   ├── b_lumen_scene_lighting.md
│   ├── c_lumen_tracing.md
│   ├── d_lumen_radiance_cache.md
│   ├── e_lumen_diffuse_gi.md
│   └── f_lumen_reflections.md
├── Reference/
│   ├── Common/
│   ├── a_SurfaceCache/
│   ├── b_SceneLighting/
│   ├── c_Tracing/
│   ├── d_RadianceCache/
│   ├── e_DiffuseGI/
│   └── f_Reflections/
└── TASK_CHECKLIST.md
```

---

## 2. 作成手順

### Step 1：ソースフォルダの調査

```
D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\{SystemName}\
```

Glob でファイル一覧を取得し、サブシステムごとにグループ化する。

**グループ化の基準:**
- 同じ機能領域（データ構造 / ライティング / トレーシング / GI / 反射 など）
- 1グループ = 5〜8ファイル程度が扱いやすい
- ヘッダ（.h）と実装（.cpp）は同じグループにまとめる
- HW RT バリアント（`*HardwareRayTracing.cpp`）はその機能グループに含める

### Step 2：TASK_CHECKLIST.md の作成

```markdown
# {SystemName} リファレンスドキュメント作成 タスクチェックリスト

## フォルダ構成（完成形）
（フォルダツリーをASCIIアートで記載）

## グループ共通（Common）— N本
| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| C-1 | `Reference/Common/ref_{system}_core.md` | `{System}.h/.cpp` | [ ] 未着手 |

## グループ a：{SubsystemName} — N本
（同様のテーブル）

## 進捗サマリ
| グループ | 合計 | 完了 | 残り |

## 作業ルール
1. 1セッションで同グループをまとめて依頼するのが効率的
2. リファレンス記事には以下を含める：
   - クラス一覧（継承・役割）
   - 主要関数一覧（引数・戻り値・役割）
   - 主要マクロ（SHADER_PARAMETER系）
   - 関連ファイルリンク
3. Detail記事の末尾に `## 関連リファレンス` セクションを追加
4. 完了したタスクは `[x]` に更新してコミット
```

### Step 3：システム概要ファイルの作成

```markdown
# {SystemName} 全体概要

- 取得日: YYYY-MM-DD
- 対象: `D:\UnrealEngine\Engine\Source\...\{SystemName}\`
- 上位: [[01_rendering_overview]]（または適切な親記事）

## {SystemName} とは
（何を解決するシステムか、従来技術との比較表）

## 全体アーキテクチャ
（Mermaid グラフで主要コンポーネントの関係を図示）

## サブシステム一覧
（各 Details 記事へのリンク一覧 + 一言説明）

## 主要ソースファイル一覧
（ファイル名 / 役割 の表）
```

### Step 4：Details 記事の作成（グループごと）

各サブシステムの概念記事。1グループ = 1ファイル。

```markdown
# {SystemName} {SubsystemName}（日本語タイトル）

- 上位: [[XX_{system}_overview]]
- 関連: [[隣接するDetailsへのリンク]]

## 概要
（このサブシステムが何をするか、全体のどこに位置するか）

## 全体フロー
（Mermaid フローチャートで処理の流れを図示）

## 主要クラス・構造体
（コードブロック + 日本語コメントで重要なフィールドを解説）

## 主要 CVar
（設定変数の一覧表）

## 関連ソースファイル
（ファイル → 役割 の表）

## 関連リファレンス    ← Step 6 で追加
（Referenceファイルへのリンク表）
```

### Step 5：Reference 記事の作成（グループごと）

1ソースファイル（またはh+cpp セット）= 1ファイル。

```markdown
# リファレンス：{FileName}.h / {FileName}.cpp

- グループ: {英字} - {SubsystemName}
- 上位: [[{对応するDetailsのID}]]
- 関連: [[他の関連Referenceのリスト]]
- ソース: `Engine/Source/Runtime/Renderer/Private/{Path}/{FileName}.h/cpp`

## 概要
（このファイルが担う役割の一言説明）

## 主要クラス・名前空間
（SHADER_PARAMETER_STRUCT の完全な定義をコードブロックで掲載）

## 主要関数
| 関数 | 引数 | 戻り値 | 説明 |

## 主要 CVar
| CVar | デフォルト | 説明 |
```

### Step 6：Details 記事に「関連リファレンス」セクションを追加

Details 記事の末尾に追記:

```markdown
## 関連リファレンス

| リファレンス | 対象ソース |
|------------|----------|
| [[ref_{system}_{subsystem}]] | `{FileName}.h/cpp` |
```

### Step 7：TASK_CHECKLIST.md の更新とコミット

グループ完了のたびに:
1. 該当グループの `[ ]` を `[x]` に変更
2. 進捗サマリ表の数値を更新
3. `git add` → `git commit`

---

## 3. ファイル命名規則

| 種類 | 命名パターン | 例 |
|------|------------|-----|
| 概要 | `XX_{system}_overview.md` | `02_lumen_overview.md` |
| Details | `{alpha}_{system}_{subsystem}.md` | `a_lumen_surface_cache.md` |
| Reference | `ref_{system}_{source_file}.md` | `ref_lumen_scene_data.md` |
| チェックリスト | `TASK_CHECKLIST.md` | `TASK_CHECKLIST.md` |

---

## 4. Obsidian WikiLink のルール

```
概要 → Details:
  [[a_lumen_surface_cache]]

Details → Details（横断リンク）:
  - 関連: [[c_lumen_tracing]] | [[d_lumen_radiance_cache]]

Reference → 親 Details:
  - 上位: [[a_lumen_surface_cache]]

Reference → 関連 Reference:
  - 関連: [[ref_lumen_mesh_cards]] | [[ref_lumen_surface_cache]]

Details → Reference（関連リファレンスセクション）:
  [[ref_lumen_scene_data]]
```

---

## 5. セッション運用のコツ

- **1セッション = 1グループ**（5〜8本）を目安にする
  - これ以上だとコンテキストが切れやすい
- グループ完了後に「次のグループに進みますか？」と確認を挟む
- セッションが切れたとき用に TASK_CHECKLIST.md を常に最新に保つ
- ソースファイルが存在しない場合はヘッダーと Lumen 等の類似システムの
  アーキテクチャパターンから内容を推定して作成してよい（主要CVar等は特に）

---

## 6. Claude への依頼テンプレート

```
{SystemName}のソースコード解析ドキュメントをLumenと同じ方式で作成してください。

対象: D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\{SystemName}\
保存先: D:\Learning\Projects\UE5\analysis\Rendering\{SystemName}\

手順:
1. ソースフォルダのファイル一覧を取得してグループ分けの案を提示
2. TASK_CHECKLIST.md を作成
3. システム概要ファイルを作成
4. グループごとに確認を取りながら進める（1セッション1グループ）

参考:
- 作成手順: D:\Learning\Projects\UE5\analysis\Rendering\HOWTO_create_system_docs.md
- Lumen の実例: D:\Learning\Projects\UE5\analysis\Rendering\Lumen\
```

---

## 7. 完成後の Obsidian 連携

Reference ファイルの frontmatter（`- 上位:`, `- 関連:`）と  
Details の `## 関連リファレンス` セクションにより、  
Obsidian のグラフビューで以下の階層が自動的に可視化される:

```
rendering_overview
  └── {system}_overview
        ├── Details/{subsystem_a}  ←→  Details/{subsystem_b}
        │     └── Reference/ref_xxx
        │     └── Reference/ref_yyy
        └── Details/{subsystem_c}
              └── Reference/ref_zzz
```
