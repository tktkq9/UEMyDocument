# Velocity Buffer シェーダー詳細

- グループ: b - Velocity
- GPU 概要: [[01_depthprepass_gpu_overview]]
- CPU 詳細: [[b_velocity_buffer]]
- リファレンス: [[ref_velocity]]

---

## 概要

**Velocity Buffer** は各ピクセルのスクリーンスペース移動ベクトルを記録する。  
スキンメッシュ・静的メッシュ（変換あり）・Nanite メッシュの3系統が対象。  
TAA・TSR・Motion Blur が VelocityBuffer を読み取りゴースト抑制・ぼかし計算に使用する。

---

## レンダリングパスの構成

```
Velocity Buffer パス
  │
  ├─ [通常メッシュ] Rasterize Pass
  │   VS: VelocityShader.usf#MainVertexShader()
  │   PS: VelocityShader.usf#MainPixelShader()
  │   RT: VelocityBuffer（R16G16F）
  │   現フレーム・前フレームのクリップ座標を補間で渡し
  │   PS で差分（Velocity.xy）を計算して書き込む
  │
  └─ [Nanite メッシュ] Compute パス
      CS: NaniteVelocity（Nanite 内部 CS）
      VisibilityBuffer から現フレーム・前フレーム位置を復元して Velocity を計算
```

---

## 入出力

### 入力（MainVertexShader）

| リソース | 説明 |
|---------|------|
| `FVertexFactoryInput` | 頂点データ |
| `View.TranslatedWorldToClip` | 現フレームの変換行列 |
| `View.PrevTranslatedWorldToClip` | 前フレームの変換行列 |

### 出力

| リソース | フォーマット | 内容 |
|---------|------------|------|
| `VelocityBuffer` | `R16G16F` | .xy = スクリーンスペース Velocity（-1〜1）|

---

## シェーダーコアロジック（VelocityShader.usf）

### MainVertexShader

```hlsl
void MainVertexShader(
    FVertexFactoryInput Input,
    out FVelocityVSToPS Output)
{
    FVertexFactoryIntermediates VFIntermediates = GetVertexFactoryIntermediates(Input);

    // 現フレームのワールド座標
    float4 WorldPos = VertexFactoryGetWorldPosition(Input, VFIntermediates);
    WorldPos.xyz += GetMaterialWorldPositionOffset(VertexParameters);

    Output.Position = mul(WorldPos, View.TranslatedWorldToClip);

    // 前フレームのワールド座標（スキンメッシュは前フレームのボーン行列を使用）
    float4 PrevWorldPos = VertexFactoryGetPreviousWorldPosition(Input, VFIntermediates);
    // PackedVelocityC に前フレームクリップ座標を格納
    Output.Interps.PackedVelocityC = mul(PrevWorldPos, View.PrevTranslatedWorldToClip);
}
```

### MainPixelShader

```hlsl
void MainPixelShader(
    FVelocityInterpsVSToPS Inputs,
    out float4 OutColor : SV_Target0)
{
    // 現在フレームのスクリーン座標（正規化）
    float2 ScreenPos = Inputs.Position.xy / Inputs.Position.w;

    // 前フレームのスクリーン座標
    float2 PrevScreenPos = Inputs.PackedVelocityC.xy / Inputs.PackedVelocityC.w;

    // Velocity = 現在 - 前フレーム（スクリーンスペース）
    float2 Velocity = ScreenPos - PrevScreenPos;

    // 深度バイアスを加えて z ファイティングを防ぐ
    // （GDepthBias = 0.001f、Reverse-Z のため正方向にバイアス）

    OutColor = float4(EncodeVelocityToTexture(Velocity), 0, 0);
}
```

### Velocity のエンコード

```hlsl
// VelocityCommon.ush
float2 EncodeVelocityToTexture(float2 Velocity)
{
    // [-1, 1] → [0, 1] にリマップ
    return Velocity * 0.5 + 0.5;
}

float2 DecodeVelocityFromTexture(float2 Encoded)
{
    return Encoded * 2.0 - 1.0;
}
```

---

## CPU 呼び出しの流れ

```
RenderVelocities()                          // VelocityRendering.cpp
  │
  ├─ EVelocityPass::Opaque → 動的オブジェクトのみ
  │   AddPass(RDG_EVENT_NAME("VelocityPass"))
  │   FOpaqueVelocityMeshProcessor::AddMeshBatch()
  │   DrawDynamicMeshCommands()
  │     VS: MainVertexShader + PS: MainPixelShader
  │     RT: VelocityBuffer（RHI_RT_VELOCITY）
  │
  ├─ EVelocityPass::Translucent → 半透明（有効時）
  │
  └─ [Nanite]
      FNaniteSceneRenderer::DrawVelocities()
      → Nanite が自前の CS で VelocityBuffer を書き出す
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `VelocityShader.usf` | VS/PS | 通常メッシュの Velocity 計算 |
| `VelocityCommon.ush` | ヘッダ | Velocity エンコード・デコード |
| `VelocityUpdate.usf` | CS | Static Mesh 変換更新後の Velocity 計算 |
