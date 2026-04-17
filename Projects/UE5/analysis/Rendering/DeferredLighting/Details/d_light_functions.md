# d: Light Function マテリアル

- 対象ファイル: `LightFunctionRendering.cpp` / `LightFunctionAtlas.h`
- 概要: [[16_deferred_lighting_overview]]

---

## 概要

**Light Function** はライトの強度や色を制御するマテリアル（テクスチャプロジェクション）。  
スポットライトの形状マスクや、光のゴッドレイ風パターン等に使用する。  
UE5 では **Light Function Atlas**（複数ライト関数をアトラス化）に対応し  
Clustered Deferred と連携してバッチ処理できる。

---

## 2つの適用経路

| 経路 | 説明 | 用途 |
|-----|------|------|
| 従来方式（`RenderLightFunctionForLight`）| ライトごとに個別テクスチャに描画 | 単発のライト関数 |
| Light Function Atlas（`LightFunctionAtlas`）| 複数ライト関数をアトラスにまとめる | Clustered Deferred と連携 |

---

## 従来方式フロー

```
RenderLight()
  │
  ├─ [Light Function あり && Atlas 非使用]
  │   RenderLightFunction()                  LightFunctionRendering.cpp:269
  │     │
  │     ├─ LightFunctionMaterial を評価
  │     ├─ テクスチャ（LightAttenuation / ShadowMask）に描画
  │     │   → ライトの ProjectionMatrix で UVs を計算
  │     └─ 描画結果をライトのアテニュエーションに乗算
  │
  └─ [Light Function なし / Atlas 使用] → スキップ
```

---

## Light Function Atlas フロー

```
LightFunctionAtlas::RenderLightFunctionAtlas()
  │
  ├─ 各ライトの LightFunction マテリアルをアトラステクスチャに描画
  │   → FLightFunctionAtlasGlobalParameters に格納
  │
  └─ [Clustered Deferred]
      AddClusteredDeferredShadingPass()
        → FLightFunctionAtlasGlobalParameters をバインド
        → シェーダー内で Atlas テクスチャをサンプル

// 使用判断
bool bUseLightFunctionAtlas = LightFunctionAtlas::IsEnabled(
    *Scene, ELightFunctionAtlasSystem::DeferredLighting);
```

---

## シェーダー側（LightFunctionAtlas.ush）

```hlsl
// Clustered Deferred シェーダー内でのサンプル
float3 LightFunctionColor = SampleLightFunctionAtlas(
    LightFunctionAtlas,
    LightFunctionAtlasUVOffset,    // アトラス内の UV オフセット
    LightFunctionAtlasUVScale,     // アトラス内の UV スケール
    WorldPosition,
    LightToWorld);                 // ライトのワールド変換行列
```

---

## `FLightFunctionSharedParameters`

```cpp
// LightRendering.h:69
class FLightFunctionSharedParameters
{
    DECLARE_TYPE_LAYOUT(FLightFunctionSharedParameters, NonVirtual);
public:
    void Bind(const FShaderParameterMap& ParameterMap);
    static FVector4f GetLightFunctionSharedParameters(
        const FLightSceneInfo* LightSceneInfo,
        float ShadowFadeFraction);
    void Set(FRHIBatchedShaderParameters& BatchedParameters,
             const FLightSceneInfo* LightSceneInfo,
             float ShadowFadeFraction) const;
private:
    LAYOUT_FIELD(FShaderParameter, LightFunctionParameters)
    // LightFunctionParameters = float4(Scale, Bias, ShadowFadeFraction, ?)
};
```

---

## 関連リファレンス

- [[ref_light_rendering]] — `FShadowedLightSceneInfo` / `FLightFunctionAtlasGlobalParameters`
- [[ref_light_params]] — `FDeferredLightUniformStruct` の `ShadowedBits` フィールド

---

## LightFunctionMaterial → テクスチャ生成 → ライット適用 詳細フロー

```
【判定】
RenderLights()
  │
  └─ for each ライスト:
      bUseLightFunctionAtlas = LightFunctionAtlas::IsEnabled(
          View, ELightFunctionAtlasSystem::DeferredLighting)
      → Atlas 有効: Atlas ルートへ
      → Atlas 無効 && Proxy.LightFunctionMaterial != null: 従来ルートへ

──────────────────────────────────────────────────────────
【従来ルート（ライストごと個別テクスチャ）】

RenderLight()
  └─ [LightFunction あり && !Atlas]
      RenderLightFunction()                        LightFunctionRendering.cpp:269
        │
        ├─ [1] LightAttenuation テクスチャを確保（R8G8B8A8 or R16F）
        │
        ├─ [2] FLightFunctionPS 描画
        │   → SetLightFunctionParameters():
        │       LightFunctionMatrix = ライストのProjection行列
        │       LightFunctionParameters = (Scale, Bias, ShadowFadeFraction, 0)
        │   → FullScreenRect で全ライスト影響範囲に PS を実行
        │   → マテリアルのテクスチャ・ノイズ関数などを評価
        │   → 結果を LightAttenuationTexture に書き込み
        │
        └─ [3] RenderLight() の PS パラメータにバインド
            DeferredLightPixelShaders.usf:
              LightAttenuationTexture.Sample(UV) → atten 係数
              LightColor *= atten

──────────────────────────────────────────────────────────
【Atlas ルート（Clustered Deferred と統合）】

[事前フェーズ] LightFunctionAtlas::RenderLightFunctionAtlas()
  │
  ├─ [1] アトラステクスチャ確保
  │   Atlas = RDGBuilder.CreateTexture(AtlasDesc, "LightFunctionAtlas")
  │   サイズ: r.LightFunctionAtlas.Size（デフォルト 256）のグリッド
  │
  ├─ [2] 各ライストの LightFunctionMaterial を Atlas のスロットに割り当て
  │   FLightFunctionAtlasItem { UVOffset, UVScale, LightSceneInfo }
  │
  ├─ [3] FLightFunctionAtlasRenderPS 描画
  │   → 各マテリアルを対応スロット（UVOffset, UVScale）に描画
  │   → FLightFunctionAtlasGlobalParameters.AtlasTexture に格納
  │
  └─ [4] FLightFunctionAtlasGlobalParameters をシェーダーにバインド
      AddClusteredDeferredShadingPass()
        → PSParameters.LightFunctionAtlas = BindGlobalParameters()

[Clustered Deferred CS 内]
ClusteredDeferredShadingPixelShader.usf:
  for each LightIndex:
    float LightFunctionValue = SampleLightFunctionAtlas(
        LightFunctionAtlas,       // Atlas テクスチャ + サンプラー
        AtlasUVOffset[LightIndex],// ライスト割り当てスロットの UV オフセット
        AtlasUVScale[LightIndex], // スロットのスケール
        WorldPosition,            // サーフェスのワールド座標
        LightToWorld[LightIndex]) // ライストの変換行列
    → LightColor *= LightFunctionValue
```

> [!note]- Atlas の再利用とキャッシュ
> 静的ライストの LightFunction マテリアルは変化しないため、
> フレーム間でアトラスを再利用する。
> ダイナミック（アニメーション付き）マテリアルは毎フレーム再描画が必要。
> `r.LightFunctionAtlas.CacheStaticLightFunctions=1` でキャッシュを有効化できる。
