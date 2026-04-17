# REF: GPUScene BuildInfo シェーダー

- グループ: c - BuildInfo
- 詳細: [[detail_build_info]]
- CPU リファレンス: [[ref_gpuscene_light_data]]
- ソース: `Engine/Shaders/Private/GPUScene/GPUSceneDataManagement.usf`  
          `Engine/Shaders/Private/LightDataUniforms.ush`  
          `Engine/Shaders/Private/SceneData.ush`

---

## LightDataBuffer アクセサ（LightDataUniforms.ush）

```hlsl
// GetLightData() - シェーダーからライトデータを取得
FLightShaderData GetLightData(uint LightId)
{
    return LightDataBuffer[LightId];
}

// GetLocalLightData() - 統合ライティングパスで使用
FLocalLightData GetLocalLightData(uint LightIndex, uint EyeIndex)
{
    return ForwardLightData.LocalLightBuffer[LightIndex];
}
```

---

## GPUScene 統計バッファ（SceneData.ush）

```hlsl
// GPUSceneParameters UB
uint NumInstances;           // 有効インスタンス総数
uint NumPrimitives;          // 有効プリミティブ総数
uint InstanceDataSOAStride;  // SOA レイアウト用ストライド

// StructuredBuffer の SOA（Structure of Arrays）レイアウト
// InstanceDataBuffer の各フィールドは個別の配列として格納
// → キャッシュ効率を上げるため（カリング時に Position だけ読む等）
```

---

## FLocalLightData CPU 構造体

```cpp
// LightRendering.h
struct FLocalLightData
{
    FVector4f LightPositionAndInvRadius;  // xyz=位置（TranslatedWorld）, w=1/Radius
    FVector4f LightColorAndIdAndFalloffExponent;  // rgb=色, w=packed(LightId, Falloff)
    FVector4f SpotAnglesAndSourceRadiusPacked;
    FVector4f LightDirectionAndShadowMask;
    FVector4f LightTangentAndSoftSourceRadius;
    FVector4f RectBarnDoorAndVolumetricScatteringIntensity;
};

// CPU 側で FLocalLightData 配列を構築
// → RHICmdList.UpdateBuffer() で LightDataBuffer に転送
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.GPUScene` | 1 | GPUScene 有効 |
| `r.GPUScene.MaxPooledUploadBufferSize` | 256MB | アップロードバッファ最大サイズ |
| `r.GPUScene.UploadEveryFrame` | 0 | 毎フレーム全 Primitive を強制 Upload |
| `r.InstanceCulling` | 1 | GPU Instance Culling 有効 |

---

> [!note]- SOA レイアウトの意味
> InstanceSceneData は Structure of Arrays（SOA）レイアウトで格納される。  
> フィールドごとに別々のバッファ（または異なるオフセット）に並べることで、  
> Instance Culling 時に「位置（AABB）だけ読む」ようなアクセスパターンのキャッシュ効率が向上する。  
> `InstanceDataSOAStride` がフィールド間のオフセット計算に使われる。
