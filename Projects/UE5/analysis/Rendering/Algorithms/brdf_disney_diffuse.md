---
name: Disney Diffuse (Burley)
description: Burley 2012 の Disney Diffuse 項の理論と UE 実装（Diffuse_Burley / Diffuse_Chan）
type: project
---

# Disney Diffuse (Burley)

- 上位: [[_algorithm_index]]
- 関連: [[brdf_ggx]] / [[brdf_smith]] / [[brdf_fresnel_schlick]] / [[brdf_lambert]]
- 採用システム: BasePass DefaultLit, Lumen 統合, Substrate
- 出典 ID: **S03**（[[_source_index]]）— Burley 2012 §5.3 Diffuse モデル

---

## 1. 何のためのアルゴリズムか

物理的に正しい **拡散反射 BRDF**。Lambert より「artist が見て自然」な結果を出すために Disney が提案。

### 素朴な手法 (Lambert) の問題

Lambert: `f_diffuse = albedo / π`

問題:
- グレイジング角度（横から見た時）で **Retro-reflection（光源方向への戻り反射）** が再現できない
- Roughness の影響を受けない（実物の布や肌は Roughness で diffuse の見え方も変わる）
- 結果として「**布や粗い肌が無機的に平坦に見える**」

### Disney Diffuse の貢献

- **Roughness を入力** に取り入れ、表面の微細凹凸が diffuse 反射に与える影響を表現
- **Retro-reflection peak** を再現（Burley 2012 で MERL データベースの計測値とフィット）
- **エネルギー保存**: NoL=NoV=0 で発散しない（Schlick 5 乗近似で減衰）

---

## 2. 理論

### 2.1 Disney Diffuse の数式

```math
f_d = \frac{\text{albedo}}{\pi} \cdot F_{D90}(L) \cdot F_{D90}(V)
```

ここで:

```math
F_{D90}(\mathbf{x}) = 1 + (F_{D90} - 1)(1 - (\mathbf{n} \cdot \mathbf{x}))^5
```

```math
F_{D90} = 0.5 + 2 \cdot \mathrm{Roughness} \cdot (\mathbf{v} \cdot \mathbf{h})^2
```

### 2.2 仕組みの解釈

- `0.5` は Lambert 基準の半分から始める
- `2 · Roughness · VoH²` は **Roughness とハーフベクトル** に依存する Retro-reflection の強さ
- Schlick 5 乗近似で **角度依存の重み** を表現（V と L 両方向の遮蔽効果を再現）

---

## 3. UE での実装

### 3.1 標準実装

`Engine/Shaders/Private/BRDF.ush:183`（旧記載 `:182` はコメント行を指していた / 関数定義は 183 行）

```hlsl
// [Burley 2012, "Physically-Based Shading at Disney"]
float3 Diffuse_Burley( float3 DiffuseColor, float Roughness, float NoV, float NoL, float VoH )
{
    float FD90 = 0.5 + 2 * VoH * VoH * Roughness;
    float FdV = 1 + (FD90 - 1) * Pow5( 1 - NoV );
    float FdL = 1 + (FD90 - 1) * Pow5( 1 - NoL );
    return DiffuseColor * ( (1 / PI) * FdV * FdL );
}
```

理論式とほぼ完全一致。`Pow5` は Schlick の 5 乗近似（[[brdf_fresnel_schlick]] と共通）。

### 3.2 比較: Lambert

`BRDF.ush:177`

```hlsl
half3 Diffuse_Lambert( half3 DiffuseColor )
{
    return DiffuseColor * (1 / PI);
}
```

入力は `DiffuseColor` のみ。NoL を呼び出し側で掛ける（cos 分布）。

### 3.3 適用箇所

`ShadingModels.ush:631`:

```hlsl
const float3 DiffuseReflection = Diffuse_Burley(GBuffer.DiffuseColor, GBuffer.Roughness, Context.NoV, AreaLight.NoL, Context.VoH);
```

DefaultLit 系 ShadingModel で標準採用。一部の SSS 系では Lambert を使う（拡散散乱が主役のため）。

### 3.4 Diffuse_Chan / Diffuse_GGX_Rough（拡張版）

`BRDF.ush` には **エネルギー補正版** や **GGX-Rough Diffuse** も実装あり:

- `Diffuse_Chan`: Chan 2018 の改良版、Burley よりエネルギー保存性向上
- `Diffuse_GGX_Rough`: 高 Roughness 時の Microfacet diffuse（GGX NDF と整合する diffuse）

UE のデフォルトは `Diffuse_Burley`。プロジェクト設定や Substrate で切り替え可能。

---

## 4. 近似・省略の差分

| 項目 | Disney オリジナル (S03) | UE 実装 | 影響 |
|------|--------------------|--------|------|
| 数式 | §5.3 そのまま | 完全一致 | なし |
| `Pow5` | 同左 | 同左 | なし |
| エネルギー補正 | Disney は SS 補正項あり | UE の `Diffuse_Burley` には未実装 | 高 Roughness で僅かにエネルギー逸失（実用問題なし） |
| Hanrahan-Krueger SSS | Disney は別途扱い | UE は SSS ShadingModel に分離 | 設計分離 |

---

## 5. パラメータと CVar

| パラメータ | デフォルト | 効果 |
|----------|----------|------|
| `Material.BaseColor` | (0.18, 0.18, 0.18) | DiffuseColor = BaseColor · (1-Metallic) |
| `Material.Roughness` | 0.5 | Disney Diffuse の `FD90` に直接影響 |

CVar 切り替えはなし（プロジェクト設定でレガシー Lambert モードに切替可能）。

---

## 6. 代替手法との比較

| 拡散モデル | 特徴 | UE 採用 |
|----------|------|--------|
| Lambert | `albedo/π` 一定 | レガシー / SSS 系 |
| Oren-Nayar (1994) | 表面凹凸を考慮、複雑 | UE には未実装 |
| **Disney/Burley (2012)** | Roughness 依存、Retro 反射 | **標準** |
| Chan (2018) | Burley + エネルギー補正 | `Diffuse_Chan` で実装あり |
| GGX-Rough Diffuse | GGX 統合 | `Diffuse_GGX_Rough` |
| Lambert + Hanrahan-Krueger SSS | SSS 専用 | Subsurface ShadingModel |

---

## 7. 参考資料

- [S03] Burley 2012, "Physically-Based Shading at Disney" — §5.3 Diffuse モデル原典、Eq.4 が核心
- Chan 2018 ブログ — Diffuse_Chan の改良提案
- 関連: [[brdf_lambert]] / [[brdf_fresnel_schlick]]（Pow5 共有）

---

## 8. 相談用フック

- **理解度チェック**: 
  - なぜ Lambert ではなく Disney Diffuse？ → §1 問題提起
  - `FD90` の意味 → §2.2、Roughness と VoH に依存する angular weight
  - Pow5 が specular と diffuse 両方で使われる理由 → Schlick の universality
- **コード深掘り候補**: 
  - `(1/PI) * FdV * FdL` の係数 1/PI の物理的意味（半球積分の正規化）
  - `Diffuse_Chan` との数値差分プロット（要 `_full.md`）
- **未読箇所**: 
  - S03 §5.5 Sheen / §5.6 ClearCoat 拡張
  - Chan 2018 の SS（Single Scattering）補正の正確な式
- **次の派生**: 
  - Substrate Layer 分離での Diffuse 扱い → [[../Substrate/...]]（未着手）
  - Lumen Surface Cache 内の Diffuse 統合 → [[lumen_surface_cache]]
