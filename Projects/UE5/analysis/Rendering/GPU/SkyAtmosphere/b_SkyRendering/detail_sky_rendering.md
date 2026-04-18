# GPU b: SkyRendering — 空のレイマーチング描画シェーダー

- シェーダー: `SkyAtmosphere.usf`（VS/PS エントリ）
- CPU 対応: [[a_sky_atmosphere]]
- 上位: [[01_skyatmosphere_gpu_overview]]

---

## 概要

SkyRendering パスは LUT ビルドパスで生成したテーブルを参照しながら、  
スカイドームをフルスクリーンレイマーチングで描画する。  
`SkyAtmosphereVS` が全画面三角形を生成し、`RenderSkyAtmosphereRayMarchingPS` が空の色を計算する。

---

## SkyAtmosphereVS（line 851）

```hlsl
void SkyAtmosphereVS(
    in uint VertexId : SV_VertexID,
    out float4 Position : SV_POSITION)
{
    float2 UV = -1.0f;
    UV = VertexId == 1 ? float2(-1.0f, 3.0f) : UV;
    UV = VertexId == 2 ? float2( 3.0f,-1.0f) : UV;

    // StartDepthZ: 空が描画される深度（不透明ジオメトリの背後に配置）
    Position = float4(UV, StartDepthZ, 1.0f);
}
```

**ポイント：**
- VertexBuffer なし（`SV_VertexID` のみ使用）
- 3頂点で全画面を覆う大三角形（Full-Screen Triangle）
- `StartDepthZ` で深度テストをパスする奥行きを指定

---

## RenderSkyAtmosphereRayMarchingPS（line 905）

```hlsl
void RenderSkyAtmosphereRayMarchingPS(
    in float4 SVPos : SV_POSITION,
    out float4 OutLuminance : SV_Target0
    // COLORED_TRANSMITTANCE_ENABLED 時は SV_Target1 も出力
    // MSAA_SAMPLE_COUNT > 1 時は SampleIndex も使用
)
{
    float2 UvBuffer = SVPos.xy * View.BufferSizeAndInvSize.zw;

    // 1. 深度からワールド位置を再構成
    // 2. カメラ → ピクセル方向のレイを計算
    // 3. SkyViewLUT をサンプリング（FASTSKY の場合）または
    //    フルレイマーチングで散乱を積分
    // 4. AerialPerspective Volume で空気遠近効果を適用
    // 5. ライトディスク（太陽・月）を描画（SOURCE_DISK_ENABLED 時）

    OutLuminance = PrepareOutput(PreExposedLuminance, Transmittance);
}
```

---

## PrepareOutput 処理

```hlsl
float4 PrepareOutput(float3 PreExposedLuminance, float3 Transmittance = 1.0)
{
    // fp10 の最大値でクランプ（NaN / Inf 防止）
    const float3 SafeLuminance = min(PreExposedLuminance, Max10BitsFloat * 0.5f);

    // グレースケール透過率をアルファに
    float GreyTransmittance = dot(Transmittance, float3(1/3, 1/3, 1/3));
    float4 Result = float4(SafeLuminance, GreyTransmittance);

    // Alpha Holdout（外宇宙では不透明 Alpha を出力）
    if (bPropagateAlphaNonReflection > 0)
        Result.a = 1 - GreyTransmittance;

    return Result;
}
```

---

## 描画モード切り替え

| マクロ | 動作 |
|--------|------|
| `FASTSKY_ENABLED = 1` | SkyViewLUT のサンプリングのみ（低コスト）|
| `FASTSKY_ENABLED = 0` | フルレイマーチング（高品質）|
| `SOURCE_DISK_ENABLED = 1` | 太陽・月のディスク描画を追加 |
| `SECOND_ATMOSPHERE_LIGHT_ENABLED = 1` | 月光等の第2ライト対応 |
| `SAMPLE_OPAQUE_SHADOW = 1` | 不透明シャドウ（大気中のシャドウ計算）|
| `SAMPLE_CLOUD_SHADOW = 1` | ボリューメトリッククラウドのシャドウ |
| `SEPARATE_MIE_RAYLEIGH_SCATTERING = 1` | Mie/Rayleigh を独立出力 |

---

## UpdateVisibleSkyAlpha

```hlsl
void UpdateVisibleSkyAlpha(in float DeviceZ, inout float4 OutLuminance)
{
    // DeviceZ == FarDepthValue（遠端クリップ）の場合 = 外宇宙を見ている
    // → Alpha を 1.0（完全不透明）に設定（スカイドームメッシュと挙動を統一）
    bool bOutputFullyOpaque = (bPropagateAlphaNonReflection > 0) && (DeviceZ == FarDepthValue);
    if (bOutputFullyOpaque) OutLuminance.a = 1.0;
}
```

---

## デバッグ / エディタ用パス

| エントリポイント | 行番号 | 用途 |
|---------------|--------|------|
| `RenderSkyAtmosphereDebugPS` | 1705 | 大気パラメータのデバッグ可視化 |
| `RenderSkyAtmosphereEditorHudPS` | 1854 | エディタ HUD オーバーレイ |

---

## 合成方法

空の描画結果は Additive ブレンドで SceneColor に合成される。  
`OutLuminance.a`（透過率）はアルファマスクとして利用され、  
不透明ジオメトリが描かれたピクセルでは空が透けないよう制御される。
