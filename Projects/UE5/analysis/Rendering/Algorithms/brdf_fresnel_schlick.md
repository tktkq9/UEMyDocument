# Schlick Fresnel 近似（フレネル項）

- 上位: [[_algorithm_index]]
- 関連: [[brdf_ggx]] / [[brdf_smith]] / [[brdf_lazarov_env]]
- 採用システム: 全 specular 計算（D・G とペアで F = D·V·F の F）
- 出典 ID: **S05**（[[_source_index]]）— Schlick 1994
- 親理論: 古典物理光学 Fresnel 1818（完全フレネル）
- 関連: **S03** Burley 2012（Disney での F0 概念整理）

---

## 1. 何のためのアルゴリズムか

**フレネル項 F**: 入射角に応じた反射率の変化を表す。

物理的事実:
- 真正面（垂直入射）から見た物体は、その素材固有の反射率 `F0` で反射する
- グレイジングアングル（90° 近く）から見ると、**ほぼ全ての素材が 100% 反射** に近づく（鏡面反射が立ち上がる）
- これは「水面を遠くから見ると鏡みたいに光る」現象の根拠

### 完全フレネル式の問題

Fresnel 1818 の正確な式は **複素屈折率を含む三角関数の積** で、リアルタイム計算には重い:

```
F(θ) = ½ · ((g-c)/(g+c))² · (1 + ((c(g+c)-1)/(c(g-c)+1))²)
ただし g = √(n² + c² - 1), c = cos(θ)
```

`F_Fresnel` (`BRDF.ush:449`) で実装はあるが、**Substrate Hair / 一部参照用**のみで標準は使わない。

### Schlick 1994 の貢献

「**5 乗近似** で十分」と示した:

```math
F(\theta) \approx F_0 + (1 - F_0)(1 - \cos\theta)^5
```

- 完全形との誤差: **典型的な誘電体で < 1%**
- 計算コスト: 1 sub + 4 mul + 1 add（劇的に軽い）

---

## 2. 理論

### 2.1 Schlick 近似式

```math
F_{\text{Schlick}}(\mathbf{v}, \mathbf{h}) = F_0 + (1 - F_0)(1 - (\mathbf{v} \cdot \mathbf{h}))^5
```

**重要**: `v · h` を使う（`v · n` ではない）。マイクロファセット理論ではハーフベクトル H 上のミラー反射を考えるため。

### 2.2 F0（垂直入射反射率）

| 素材 | F0 (sRGB) |
|------|-----------|
| 水 | 0.02 |
| プラスチック | 0.04 |
| ガラス | 0.04 |
| ダイヤモンド | 0.17 |
| 金属（金） | (1.00, 0.71, 0.29) |
| 金属（銀） | (0.95, 0.93, 0.88) |
| 金属（銅） | (0.95, 0.64, 0.54) |

### 2.3 メタリック ワークフロー

UE は Disney 流の **Metallic + BaseColor** 入力:

```
F0 = lerp(0.04, BaseColor, Metallic)
DiffuseColor = BaseColor * (1 - Metallic)
```

- Metallic = 0: 誘電体、F0 = 0.04（誘電体共通の近似値）
- Metallic = 1: 金属、F0 = BaseColor（金属の色は反射色そのもの）

これにより **PBR 入力をたった 3 パラメータ (BaseColor, Metallic, Roughness)** に圧縮。

### 2.4 F90

物理的には常に 1.0（完全反射）。一部の **Disney 拡張** や **Cloth Sheen** では F90 を別途指定して柔らかい光沢を表現:

```math
F = F_0 + (F_{90} - F_0)(1 - \cos\theta)^5
```

UE の `F_Schlick(F0, F90, VoH)` オーバーロードがこれ。

---

## 3. UE での実装

### 3.1 標準 `F_Schlick`

`BRDF.ush:423`

```hlsl
// [Schlick 1994]
float3 F_Schlick( float3 SpecularColor, float VoH )
{
    float Fc = Pow5( 1 - VoH );  // 1 sub, 3 mul
    
    // Anything less than 2% is physically impossible and is instead considered to be shadowing
    return saturate( 50.0 * SpecularColor.g ) * Fc + (1 - Fc) * SpecularColor;
}
```

理論との対応:
- `Fc = (1 - VoH)^5` = 理論式の `(1 - cos θ)^5`
- `(1 - Fc) * SpecularColor` = `F0 · (1 - Fc)`
- `saturate(50.0 * SpecularColor.g) * Fc` = `F90 · Fc` だが **F90 を実質 0 にする「2% 閾値ハック」**

### 3.2 「2% 閾値ハック」の意味

```hlsl
saturate( 50.0 * SpecularColor.g )
```

- SpecularColor.g（緑成分）が 0.02 以上なら 1.0 にクランプ → 通常通り F90=1
- 0.02 未満なら線形に 0 へ → グレイジングのハイライトを完全に消す

**目的**: Specular = 0 のマテリアル設定を「シャドウ表現用」として機能させる（古い PBR 実装では F90 を強制 1 にしていたが、UE は 「0 を信じる」ポリシー）。

### 3.3 F0/F90 オーバーロード

`BRDF.ush:432`

```hlsl
float3 F_Schlick(float3 F0, float3 F90, float VoH)
{
    float Fc = Pow5(1 - VoH);
    return F90 * Fc + (1 - Fc) * F0;
}
```

数学的には Disney/Cloth で F90 を独立指定する場合の正確な実装。

### 3.4 Adobe F82（拡張版）

`BRDF.ush:438` `F_AdobeF82` — Kutz 2021 "Novel aspects of the Adobe Standard Material" 由来。  
グレイジングよりわずかに手前（θ ≈ 82°）にディップを入れる **3 パラメータ Fresnel**。金属の色味再現性を上げる用途。

### 3.5 完全フレネル `F_Fresnel`

`BRDF.ush:449` — 上記の物理式を実装。Hair / 特殊 BSDF 用。

---

## 4. 近似・省略の差分

| 項目 | 完全フレネル | UE Schlick | 影響 |
|------|------------|----------|------|
| 計算式 | 複素屈折率含む三角関数 | 5 乗の多項式 | 典型誘電体で 1% 以内 |
| 入力パラメータ | 屈折率 n | F0 (= ((n-1)/(n+1))²) | F0 ↔ n は 1 対 1 |
| 金属対応 | 複素 n | RGB の F0 で代用 | 物理的には正しくない近似だが視覚的に十分 |
| F90 | 常に 1 | 2% 閾値ハック or 明示 F90 | アーティスト制御の柔軟性 |

---

## 5. パラメータと CVar

| パラメータ | デフォルト | 効果 |
|----------|----------|------|
| `Material.SpecularColor` | (0.04, 0.04, 0.04) | F0、Metallic で BaseColor とブレンド |
| `Material.Specular` | 0.5 | 誘電体の F0 を `0.08 · Specular` で算出 |

---

## 6. 代替手法との比較

| 手法 | コスト | 精度 | UE 採用 |
|------|------|------|--------|
| 完全フレネル | 重い | ◎ | `F_Fresnel`（特殊 BSDF のみ） |
| **Schlick 1994** | 軽い | ◯（誤差 < 1%） | **`F_Schlick` 標準** |
| Schlick (F0, F90) | 同上 | ◯+ | `F_Schlick(F0, F90)` オーバーロード |
| Adobe F82 (2021) | 軽い | ◎（金属向上） | `F_AdobeF82`（Substrate） |
| Gulbrandsen 2014 (Artist friendly) | 軽い | ◯ | 未採用 |

---

## 7. 参考資料

- [S05] Schlick 1994, "An Inexpensive BRDF Model for Physically-Based Rendering" — Fresnel 5 乗近似の原典
- [S03] Burley 2012 §5.4 — F0 = 0.04 採用根拠
- Kutz et al. 2021, "Novel aspects of the Adobe Standard Material" — `F_AdobeF82` の出典
- 関連: [[brdf_ggx]] / [[brdf_smith]]

---

## 8. 相談用フック

- **理解度チェック**: 
  - F0 と F90 の違い → §2.2, §2.4
  - なぜ「メタリック」と「ベースカラー」だけで PBR が成立するか → §2.3
  - 「2% 閾値ハック」の意味 → §3.2
- **コード深掘り候補**: 
  - `Pow5(1-VoH)` のコンパイル後命令数
  - `saturate(50 * SpecularColor.g)` を g チャンネルで判定する理由（人の目の感度ピーク）
- **未読箇所**: 
  - Adobe F82 の `K = 49/46656` 定数導出（コメントで「constant folding from CosThetaMax=1/7」とある）
- **次の派生**: 
  - 多層 BSDF (ClearCoat) の二重 Fresnel → [[../BasePass/Details/...]]
  - Substrate での Fresnel 統合 → [[../Substrate/...]]
