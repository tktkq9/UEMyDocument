# Lumen コード実行フロー 追記タスクリスト

**方針**: 概要・Details 各ファイルに「どこから処理が始まり、どの関数・クラスが呼ばれるか」を追記する  
**追記形式**: `## コード実行フロー` セクション（エントリポイント → 段階的コールチェーン + コードスニペット）  
**作業開始日**: 2026-04-11  
**優先度**: 概要 → a → b → c → d → e → f の順

---

## 追記内容の種類

各ファイルに以下を追加する:

| 追記項目 | 形式 | 説明 |
|---------|------|------|
| **エントリポイント** | コードブロック | `FDeferredShadingSceneRenderer::Render()` 等の呼び出し元を明示 |
| **コールチェーン** | 番号付きステップ + `cpp` スニペット | 関数 → 関数 の連鎖を順番に追う |
| **分岐条件** | インラインコードブロック | SW RT / HW RT など重要な分岐を `if` ブロックで示す |
| **関与クラス一覧** | テーブル | そのフローで登場する主要クラス・構造体の一覧 |
| **Obsidian リンク** | `[[ref_xxx]]` | 各ステップで呼ばれる関数の Reference へのリンク |

---

## 追記フォーマット（標準テンプレート）

```markdown
## コード実行フロー

### エントリポイント

```cpp
// FDeferredShadingSceneRenderer::Render() (DeferredShadingRenderer.cpp)
RenderLumen(GraphBuilder, SceneTextures, ...);
  │
  └─ UpdateLumenScene(...)        ← [[ref_lumen_scene]]
       └─ ...
```

### フロー詳細

1. **{ステップ名}** — 説明
   ```cpp
   // {ファイル名}.cpp
   ReturnType Function(Params...);
   ```
   - 条件: `{CVar or 条件式}`
   - 参照: [[ref_xxx]]

2. **{ステップ名}** — 説明
   ...

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|------------|--------|------|
| `ClassName` | `File.cpp` | 説明 |
```

---

## タスク一覧

### 概要ファイル（1件）

- [x] `02_lumen_overview.md` — Lumen 全体エントリポイント〜サブシステム呼び出しの大局フロー

**追記内容**:
- `FDeferredShadingSceneRenderer::Render()` → `RenderLumen()` → 各サブシステムへの全体コールグラフ
- フレーム内での Lumen 処理の実行順（A〜F の各ステップを関数レベルで補足）
- SW RT / HW RT の切り替えポイント

---

### a: Surface Cache（1件）

- [x] `Details/a_lumen_surface_cache.md` — Surface Cache 更新フロー

**追記内容**:
- `RenderLumen()` → `UpdateLumenScene()` → `UpdateLumenSurfaceCache()` のコールチェーン
- Card のダーティ検出 → キャプチャ → Atlas 書き込みの流れ
- `FLumenPrimitiveGroup` → `FLumenMeshCards` → `FLumenCard` の生成フロー
- GPU Driven 更新パスとの分岐

---

### b: Scene Lighting（1件）

- [ ] `Details/b_lumen_scene_lighting.md` — Surface Cache ライティングフロー

**追記内容**:
- `UpdateLumenScene()` → `RenderLumenSceneLighting()` のコールチェーン
- Direct Lighting 注入パス（SW / HW RT 分岐含む）
- Radiosity 伝播パス（Gather → Scatter のパイプライン）
- `FinalLightingAtlas` が完成するまでの流れ

---

### c: Tracing（1件）

- [ ] `Details/c_lumen_tracing.md` — トレースパス呼び出しフロー

**追記内容**:
- Screen Probe / 反射からのトレース呼び出しの共通入口
- Mesh SDF → Global SDF → Far Field の多段フォールバック実行フロー
- HW RT（Inline / RGS）への切り替えフロー
- `CompactTraces()` を軸にしたコンパクト化 → Dispatch の連鎖

---

### d: Radiance Cache（1件）

- [ ] `Details/d_lumen_radiance_cache.md` — Radiance Cache 更新フロー

**追記内容**:
- `RenderLumenDiffuseIndirect()` / `RenderLumenReflections()` から `UpdateLumenRadianceCache()` への呼び出し
- Probe の割り当て → トレース → フィルタリング → アトラス書き込みのパイプライン
- クリップマップ段別スクロール更新の仕組み
- Screen Probe / Reflections からの参照（補間）フロー

---

### e: Diffuse GI（1件）

- [ ] `Details/e_lumen_diffuse_gi.md` — Diffuse GI 実行フロー

**追記内容**:
- `RenderLumenDiffuseIndirect()` エントリから `RenderScreenProbeGather()` までのコールチェーン
- Screen Probe 配置 → トレース → フィルタリング → Gather → テンポラル蓄積の全ステップ
- Importance Sampling 更新の挿入タイミング
- Short Range AO の呼び出し位置

---

### f: Reflections（1件）

- [ ] `Details/f_lumen_reflections.md` — Reflections 実行フロー

**追記内容**:
- `RenderLumenReflections()` エントリからトレース・ReSTIR・Compose までの全コールチェーン
- SW RT（`TraceReflections()`）と HW RT（`RenderLumenHardwareRayTracingReflections()`）の分岐フロー
- ReSTIR パイプライン（初期サンプリング → テンポラル → 空間）の呼び出し順
- Front Layer Translucency / Ray Traced Translucency の呼び出し位置

---

## 進捗サマリ

| ファイル | 状態 |
|---------|------|
| `02_lumen_overview.md` | [x] 完了 |
| `a_lumen_surface_cache.md` | [x] 完了 |
| `b_lumen_scene_lighting.md` | [ ] 未着手 |
| `c_lumen_tracing.md` | [ ] 未着手 |
| `d_lumen_radiance_cache.md` | [ ] 未着手 |
| `e_lumen_diffuse_gi.md` | [ ] 未着手 |
| `f_lumen_reflections.md` | [ ] 未着手 |

**合計**: 7 ファイル  
**完了数**: 2 / 7

---

## 作業ルール

1. **1セッション = 1〜2ファイル** が目安（ソース調査 + 追記）
2. 各フローはソースコード（`D:\UnrealEngine\Engine\Source\`）を実際に確認してから記述する
3. 関数名・引数は実際のシグネチャに合わせる（推測で書かない）
4. 重要な分岐は `if` ブロックのスニペットで明示する
5. 各ステップには対応 Reference ファイルへの `[[ref_xxx]]` リンクを付ける
6. 完了したタスクは `[x]` に変更してコミット

---

## Claude への依頼テンプレート

```
Lumenのコード実行フロー追記をお願いします。

対象チェックリスト: D:\Learning\Projects\UE5\analysis\Rendering\Lumen\TASK_CHECKLIST_FLOW.md
フォーマット: 同ファイルの「追記フォーマット（標準テンプレート）」を使用
ソースコード: D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\Lumen\

{ファイル名}をお願いします。
```
