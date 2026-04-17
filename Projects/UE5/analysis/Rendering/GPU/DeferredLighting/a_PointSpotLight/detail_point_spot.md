# Point / Spot Light Deferred Shading シェーダー詳細

- グループ: a - PointSpotLight
- GPU 概要: [[01_deferred_lighting_gpu_overview]]
- CPU 詳細: [[b_local_lights]]
- リファレンス: [[ref_point_spot]]

---

## 概要

**Clustered Deferred Shading** は Light Grid（クラスタリング済みのローカルライトリスト）を使って  
各ピクセルに影響するローカルライト（Point/Spot/Rect）を全画面パスで一括評価する。  
ライトの数が多くてもクラスターごとに処理するため、全ライトを全ピクセルに適用する Naive 方式より高効率。

---

## レンダリングパスの構成

```
Clustered Deferred Shading
  │
  ├─ VS: ClusteredDeferredShadingVertexShader.usf（フルスクリーントライアングル）
  └─ PS: ClusteredDeferredShadingPixelShader.usf#ClusteredShadingPixelShader()
      1. GBuffer デコード（GBufferA/B/C/D から法線・BaseColor・Roughness・ShadingModelID）
      2. LightGrid からクラスター内のローカルライトリストを取得
      3. 各ローカルライトに対して:
           DistanceAttenuation + SpotAngle減衰 + IES プロファイル
           → BRDF（GGX + Lambert）計算
           → シャドウマスク（Shadow Map / VSM）乗算
      4. 全ライトの寄与を加算
      5. SceneColor にアディティブブレンド
```

---

## 入出力

### 入力

| リソース | 説明 |
|---------|------|
| `GBufferA/B/C/D/E` | マテリアル属性（法線・BaseColor・Roughness 等）|
| `LightGrid`（Light Grid Tiling）| クラスター別ローカルライトインデックスリスト |
| `LightDataBuffer` | 全ローカルライトのシェーダーパラメータ |
| `SceneDepth` | ワールド座標復元用 |

### 出力

| リソース | フォーマット | 内容 |
|---------|------------|------|
| `SceneColor` | `R11G11B10F` | ローカルライトの Diffuse + Specular 加算値 |

---

## シェーダーコアロジック

```hlsl
// ClusteredDeferredShadingPixelShader.usf
void ClusteredShadingPixelShader(
    float4 SvPosition : SV_POSITION,
    out float4 OutColor : SV_Target0)
{
    float2 ScreenUV = SvPositionToBufferUV(SvPosition);
    float SceneDepth = ConvertFromDeviceZ(LookupDeviceZ(ScreenUV));

    // GBuffer デコード
    FGBufferData GBuffer = DecodeGBufferData(GBufferA, GBufferB, GBufferC, ..., ScreenUV);
    float3 WorldPosition = ReconstructTranslatedWorldPosition(ScreenUV, SceneDepth);

    // Light Grid のクラスターインデックス計算
    uint GridIndex = ComputeLightGridCellIndex(SvPosition.xy, SceneDepth);

    // クラスター内のローカルライトを列挙
    uint NumLights = LightGridData.NumLights[GridIndex];
    float3 TotalLighting = 0;

    for (uint i = 0; i < NumLights; i++)
    {
        uint LightIndex = LightGridData.LightIndices[GridIndex][i];
        FLightShaderData LightData = GetLightData(LightIndex);

        // 距離・角度の減衰
        float3 ToLight = LightData.TranslatedWorldPosition - WorldPosition;
        float Attenuation = GetLocalLightAttenuation(LightData, ToLight, ...);

        // BRDF（GGX Specular + Lambert Diffuse）
        FDirectLighting Lighting = EvaluateBRDF(GBuffer, WorldPosition, ToLight, LightData, Attenuation);

        // シャドウマスク
        float ShadowFactor = GetLocalLightShadow(LightData, WorldPosition, ...);

        TotalLighting += (Lighting.Diffuse + Lighting.Specular) * ShadowFactor;
    }

    OutColor = float4(TotalLighting, 0);
}
```

### Substrate 対応（FastPath / ComplexPath）

```hlsl
// SUBSTRATE_TILETYPE に応じて3種類のパスをコンパイル
// FastPath (SUBSTRATE_FASTPATH=1): 単純なシングル BSDF メッシュ
// SinglePath: 1 BSDF + 追加レイヤー
// ComplexPath: 任意の BSDF ツリー（完全なループ処理）
```

---

## CPU 呼び出しの流れ

```
RenderClusteredDeferredLighting()            // ClusteredDeferredShading.cpp
  │
  ├─ ComputeLightGrid() → LightGrid の構築（ライトをクラスターに割り当て）
  ├─ AddPass(RDG_EVENT_NAME("ClusteredDeferredShading"))
  │   VS: ClusteredDeferredShadingVertexShader
  │   PS: ClusteredShadingPixelShader（Substrate Tile Type 別）
  │   RT: SceneColor（Additive Blend）
  └─ [Substrate 有効時] Tile Type 別に複数パスに分割
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `ClusteredDeferredShadingPixelShader.usf` | PS | Clustered Deferred ライティング本体 |
| `DeferredLightingCommon.ush` | ヘッダ | BRDF・減衰・シャドウマスクのユーティリティ |
| `LightGridCommon.ush` | ヘッダ | LightGrid アクセス・クラスターインデックス計算 |
| `LightDataUniforms.ush` | ヘッダ | `GetLightData()` |
