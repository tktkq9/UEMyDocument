---
name: SSR (Stachowiak Screen Space Reflection)
description: スクリーンスペース反射の理論と UE 実装（HZB march + temporal）
type: project
---

# SSR — Screen Space Reflection

- 上位: [[_algorithm_index]]
- 関連: [[ss_ssgi]] / [[ss_ssao]] / [[lumen_hw_rt]] / [[brdf_ggx]]
- 採用システム: 反射のフォールバック（Lumen Reflections / RT Reflections 不使用時）
- 出典:
  - **S41**: Stachowiak 2015 SIGGRAPH "Stochastic Screen-Space Reflections" (Killzone/Frostbite 系)
  - 影響元: Sousa 2011 (Crysis 2 SSR), McGuire & Mara 2014 "Efficient GPU Screen-Space Ray Tracing"
  - UE 実装: `ScreenSpaceRayTracing.cpp` / `SSRT/*.usf`

---

## 1. 何のためのアルゴリズムか

ガラス・床・水面の反射を **追加ジオメトリパス無しで** 取得したい。
- Reflection Capture: 静的、視差不正
- Planar Reflection: 平面ジオメトリのみ
- Raytracing: GPU 重い

### SSR の原理

「反射ベクトルを画面内で march し、HZB で深度交差を取る」だけで Scene Color から反射を読める。完全 image-space で動的物体に追従。

### 制限

- 画面外の対象は反射不可（off-screen は cubemap fallback）
- 画面内で隠れた面（裏側）は不可
- glossy 反射は importance sample × N → ノイズ

---

## 2. 理論

### 2.1 反射ベクトル

```
N = normalize(GBufferNormal)
V = normalize(View - WorldPos)
R = reflect(-V, N)
```

GGX glossy なら importance sample で半球内サンプル方向 H を取り `R = reflect(-V, H)`。

### 2.2 Screen Space Ray Cast（McGuire 2014 / Stachowiak 2015）

```
1. ワールド R を screen 座標に投影 → RayStartScreen, RayStepScreen
2. 画面端で clip：GetStepScreenFactorToClipAtScreenEdge
3. HZB を mip ごとに march（near hit ほど精細 mip、遠は粗 mip）
4. depth 交差を二分探索で精緻化
5. ヒット座標から SceneColor をサンプル
```

`SSRTRayCast.ush:46-56`:
```hlsl
float GetStepScreenFactorToClipAtScreenEdge(float2 RayStartScreen, float2 RayStepScreen)
{
    const float RayStepScreenInvFactor = 0.5 * length(RayStepScreen);
    const float2 S = 1 - max(abs(RayStepScreen + RayStartScreen * RayStepScreenInvFactor) - RayStepScreenInvFactor, 0.0f) / abs(RayStepScreen);
    const float RayStepFactor = min(S.x, S.y) / RayStepScreenInvFactor;
    return RayStepFactor;
}
```

### 2.3 HZB Hierarchical Trace

線形 march は遠距離で 100+ ステップ要する。HZB（Hi-Z）の mip 階層で:
- 高 mip（粗深度）で「絶対衝突しない領域」をスキップ
- 衝突候補で mip を下げて精緻化

数十ステップで kilo-pixel range をカバー。

### 2.4 Stochastic SSR（Stachowiak 2015）

GGX glossy 反射のため:
1. **Importance Sample H** をピクセルごとランダム
2. 1 ピクセル = 1 ray のみ（spp=1）
3. **Spatial Reuse + Temporal Reuse** でノイズ補完
4. Final blur が roughness に応じてカーネル変える

ReSTIR 以前の最先端 stochastic 手法。

### 2.5 Compare Tolerance

`FSSRTRay::CompareTolerance` でレイ深度と HZB 深度の許容差を設定。slope-aware（傾斜面で大きく取る）。

### 2.6 Half-Res / Tile Classification

- `r.SSR.HalfResSceneColor=1`: SceneColor を半解像でサンプル → 帯域削減
- `SSRTTileClassification.usf`: roughness 閾値で「鏡面 / glossy / 不要」を分類しタイル単位で起動

### 2.7 Temporal Reproject

`r.SSR.Temporal=1`: 前フレーム履歴を motion vector で再投影し変動を平滑化。Stochastic SSR ではほぼ必須。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `ScreenSpaceRayTracing.cpp` | SSR / SSGI 共通 pass セットアップ、CVar |
| `ScreenSpaceRayTracing.h` | API |
| `SSRT/SSRTRayCast.ush` | レイキャスト本体 |
| `SSRT/SSRTReflections.usf` | SSR メインシェーダ |
| `SSRT/SSRTPrevFrameReduction.usf` | 前フレーム scenecolor の縮小 |
| `SSRT/SSRTTileClassification.usf` | タイル振り分け |

### 3.2 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.SSR.Quality` | 3 | 0=off / 1=low / 2=mid / 3=high glossy / 4=very high |
| `r.SSR.HalfResSceneColor` | 0 | 入力 SceneColor を半解像化 |
| `r.SSR.Temporal` | 0 | 履歴ブレンド |
| `r.SSR.Stencil` | 0 | stencil prepass |
| `r.SSR.Compute` | 0 | 0=PS, 1=CS, 2=async CS |

### 3.3 PostProcessVolume Settings

- `ScreenSpaceReflectionIntensity` (0-100)
- `ScreenSpaceReflectionQuality` (0-100)
- `ScreenSpaceReflectionMaxRoughness`（これ以上 rough なら SSR 無効）

### 3.4 サンプルバッチ

`SSRT_SAMPLE_BATCH_SIZE = 4`（`SSRTRayCast.ush:6`）でレイを 4 本まとめて HZB sample → ALU/TEX 比改善。

---

## 4. 近似・省略の差分

| 項目 | 理想 RT | UE SSR | 影響 |
|------|--------|-------|------|
| Off-screen 反射 | 取得可 | 取得不可 → cubemap fallback | カメラ外オブジェクトで切替段差 |
| 裏面 / 隠面 | 完全 | 取得不可 | キャラの裏側が消える |
| サンプル数 | 任意 | spp=1 + spatial+temporal | rough で残ノイズ |
| Compare 精度 | 完全 | tolerance + 二分探索 | 微小段差で artifact |
| BRDF | 完全 GGX | 近似 importance | ハイライト形状ずれ |

---

## 5. パラメータと CVar

§3.2 にまとめ済み。glossy 強化は `r.SSR.Quality=4`、軽量化は `r.SSR.HalfResSceneColor=1`。

---

## 6. 代替手法との比較

| 手法 | 領域 | 速度 | UE 採用 |
|------|------|------|--------|
| Reflection Capture | 静的キャプチャ | 高速 | フォールバック |
| Planar Reflection | 平面ジオメトリ専用 | 中 | 水面・床 |
| **SSR (Stachowiak 2015)** | **画面内動的** | **中** | **標準フォールバック** |
| Lumen Reflections | SS + SDF + HW RT 統合 | 中〜高 | UE5 標準 |
| HW RT Reflections | DXR / RTX | 高 | High-end |
| SSGI（拡散） | 拡散も同枠 | 中 | → [[ss_ssgi]] |

### Lumen 時代の SSR

Lumen Reflections は内部で SSR をプライマリ手段として呼び出し、SS 失敗時に SDF / HW RT にフォールバック。SSR 単体使用は Lumen 無効時または明示的にコスト下げたい時。

---

## 7. 参考資料

- [S41] Stachowiak 2015 SIGGRAPH "Stochastic Screen-Space Reflections"
- McGuire & Mara 2014 "Efficient GPU Screen-Space Ray Tracing"
- Sousa 2011 GDC "Secrets of CryENGINE 3 Graphics Technology"
- 関連: [[ss_ssgi]] / [[lumen_hw_rt]] / [[brdf_ggx]]

---

## 8. 相談用フック

- **理解度チェック**:
  - HZB march の利点 → 高 mip でスキップ、低 mip で精緻化（log スケール探索）
  - Stochastic SSR の sp=1 でも見える理由 → spatial+temporal で N サンプル相当
  - SSR の本質的限界 → 画面外と裏面
- **コード深掘り候補**:
  - `SSRTRayCast.ush` の HZB march loop
  - `SSRTReflections.usf` の importance sample / spatial reuse
  - `SSRTTileClassification.usf` の振り分け閾値
- **未読箇所**:
  - 二分探索精緻化の終了条件
  - VRS（Variable Rate Shading）連携
  - Substrate での SSR 評価
- **次の派生**:
  - 拡散 GI 用 SSR → [[ss_ssgi]]
  - SDF / HW RT へのフォールバック → [[lumen_hw_rt]]
  - GGX BRDF → [[brdf_ggx]]
