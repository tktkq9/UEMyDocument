# GPUScene BuildInfo シェーダー詳細

- グループ: c - BuildInfo
- GPU 概要: [[01_gpuscene_gpu_overview]]
- CPU 詳細: [[c_gpuscene_build]]
- リファレンス: [[ref_build_info]]

---

## 概要

**SceneBuildInfo** と **LightData** の GPU バッファへの転送・インデックス構築を行う。  
ライトデータ（Point/Spot/Rect の位置・色・減衰パラメータ）が  
`LightDataBuffer` に格納され、各シェーダーから `GetLightData(LightId)` でアクセスされる。

---

## レンダリングパスの構成

```
BuildInfo Upload
  │
  ├─ [Light Data 転送]
  │   CPU: FLocalLightData → GPU: LightDataBuffer（StructuredBuffer）
  │   毎フレーム変更があったライトのみ差分転送
  │
  └─ [SceneBuildInfo 更新]
      SceneUniformBuffer の各フィールドを更新
      → GPUScene.NumInstances / NumPrimitives 等のカウント更新
```

---

## 入出力

### 入力

| リソース | 説明 |
|---------|------|
| `FLocalLightData[]` | CPU 側のローカルライトデータ配列 |
| `ChangedLightIds` | 更新されたライト ID リスト |

### 出力

| リソース | フォーマット | 内容 |
|---------|------------|------|
| `LightDataBuffer` | `StructuredBuffer<FLightShaderData>` | 全ローカルライトのシェーダーパラメータ |
| `GPUScene.NumInstances` | `uint` | 有効インスタンス総数 |

---

## FLightShaderData 主要フィールド

```hlsl
// LightDataUniforms.ush
struct FLightShaderData
{
    float3 TranslatedWorldPosition;  // ライト位置（Translation 済み）
    float  InvRadius;                // 影響半径の逆数

    float3 Color;                    // ライト色（強度含む）
    float  FalloffExponent;          // 減衰指数（レガシー）

    float3 Direction;                // スポットライト方向
    float  SpecularScale;            // スペキュラースケール

    float2 SpotAngles;               // CosInner, CosOuter（スポットライト）
    float  SourceRadius;             // 光源半径（区域光）
    float  SoftSourceRadius;         // ソフト半径

    uint   LightingChannelMask;      // ライティングチャンネル
    uint   Flags;                    // キャストシャドウ等のフラグ
    // ...
};
```

---

## CPU 呼び出しの流れ

```
FScene::UpdateAllPrimitiveSceneInfos()         // Scene.cpp
  │
  ├─ ライトデータの構築（FLocalLightData 配列）
  │
  └─ FGPUScene::UpdateLightData()
      変更されたライトのデータを Upload Buffer に書き込み
      AddPass(RDG_EVENT_NAME("GPUSceneLightDataUpload"))
      → LightDataBuffer を RHI UploadBuffer でコピー

RenderForward() / DeferredLighting 等でバインド:
  PassParameters->LightDataBuffer = GPUScene.LightDataBuffer;
  → GetLightData(LightId) でシェーダーからアクセス
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `GPUScene/GPUSceneDataManagement.usf` | CS | Instance/Primitive データ管理 |
| `LightDataUniforms.ush` | ヘッダ | `FLightShaderData` / `GetLightData()` |
| `SceneData.ush` | ヘッダ | `GetPrimitiveData()` / `GetInstanceSceneData()` |
