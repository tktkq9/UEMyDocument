# ref: FDBufferTextures / FDBufferData / ApplyDBufferDecal

- 対象ファイル: `DecalRenderingShared.cpp` / `BasePassRendering.cpp` / `SceneTextures.h`
- 概要: [[15_decals_overview]]

---

## FDBufferTextures（SceneTextures.h）

```cpp
// DBuffer デカールのレンダーターゲット管理構造体
struct FDBufferTextures
{
    // DBufferA: BaseColor + マスク
    FRDGTextureRef DBufferA = nullptr; // R8G8B8A8_UNORM
    // DBufferB: Normal(xy) + Smoothness
    FRDGTextureRef DBufferB = nullptr; // R8G8B8A8_UNORM
    // DBufferC: Metallic/Specular/Roughness + マスク
    FRDGTextureRef DBufferC = nullptr; // R8G8B8A8_UNORM
    // DBufferMask: ステンシルマスク（オプション）
    FRDGTextureRef DBufferMask = nullptr; // R8_UNORM

    bool IsValid() const
    {
        return DBufferA != nullptr && DBufferB != nullptr && DBufferC != nullptr;
    }
};
```

---

## DBuffer テクスチャのデフォルト値

```
クリア値（DBuffer に何も書き込まれていない状態）:
  DBufferA: (0, 0, 0, 1)      // BaseColor = 黒, OpacityMask = 1（デカールなし）
  DBufferB: (0.5, 0.5, 0, 1)  // Normal = 中立（0方向）, Smoothness = 0
  DBufferC: (0, 0.5, 1, 1)    // Metallic = 0, Roughness = 0.5, Specular = 1

ピクセルシェーダーでのデコード（DeferredDecals.ush）:
  OpacityMask = 1 - DBufferA.a  // 1.0 = デカールで完全上書き
  BaseColor = DBufferA.rgb
  Normal.xy  = DBufferB.rg * 2 - 1  // [0,1] → [-1,1]
  Roughness  = DBufferC.b
  Metallic   = DBufferC.r
  Specular   = DBufferC.g
```

---

## FOpaqueBasePassUniformParameters（DBuffer 関連フィールド）

```cpp
BEGIN_UNIFORM_BUFFER_STRUCT(FOpaqueBasePassUniformParameters, )
    // ... 他フィールド ...
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DBufferATexture)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DBufferBTexture)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DBufferCTexture)
    SHADER_PARAMETER_SAMPLER(SamplerState, DBufferATextureSampler)
    SHADER_PARAMETER_SAMPLER(SamplerState, DBufferBTextureSampler)
    SHADER_PARAMETER_SAMPLER(SamplerState, DBufferCTextureSampler)
    SHADER_PARAMETER(uint32, DBufferDecalMask) // デカールが存在するかのビットフラグ
END_UNIFORM_BUFFER_STRUCT()
```

---

## BasePass での ApplyDBufferDecal（DeferredDecals.ush）

```hlsl
// BasePassPixelShader.usf 内での適用処理

// DBuffer テクスチャから FDBufferData を読み取る
FDBufferData DBufferData = GetDBufferData(ScreenUV, BasePass.DBufferATexture, ...);

// FDBufferData 構造体
struct FDBufferData
{
    float3 PreMulBaseColor;           // デカールの BaseColor × OpacityMask
    float  BaseColorOpacity;          // OpacityMask
    float3 PreMulWorldNormal;         // Normal × OpacityMask
    float  NormalOpacity;
    float  PreMulRoughness;
    float  PreMulMetallic;
    float  PreMulSpecular;
    float  RoughnessOpacity;
};

// マテリアルへの適用
void ApplyDBufferData(
    FDBufferData DBuffer,
    inout float3 BaseColor,
    inout float3 Normal,
    inout float Roughness,
    inout float Metallic,
    inout float Specular)
{
    BaseColor = BaseColor * (1 - DBuffer.BaseColorOpacity) + DBuffer.PreMulBaseColor;
    Normal    = normalize(Normal * (1 - DBuffer.NormalOpacity) + DBuffer.PreMulWorldNormal);
    Roughness = Roughness * (1 - DBuffer.RoughnessOpacity) + DBuffer.PreMulRoughness;
    Metallic  = Metallic  * (1 - DBuffer.RoughnessOpacity) + DBuffer.PreMulMetallic;
    Specular  = Specular  * (1 - DBuffer.RoughnessOpacity) + DBuffer.PreMulSpecular;
}
```

---

## DBuffer 生成タイミング

```
InitViews()
  └─ FDecalVisibilityTaskData::Launch()  // 非同期デカール可視判定

PreBasePass フェーズ:
  RenderDBufferDecals()
    → BeforeBasePass ステージのデカールのみ描画
    → DBufferA/B/C に書き込み

BasePass:
  FOpaqueBasePassUniformParameters に DBuffer テクスチャをバインド
  各メッシュの PS 内で ApplyDBufferData() が呼ばれる
  → 静的ライティング（ライトマップ）にも反映される

注意: r.DBuffer=0 の場合は DBuffer テクスチャが作成されず
  DBuffer デカールは BeforeLighting ステージにフォールバック
```

---

## 関連リファレンス

- [[ref_deferred_decal]] — `FDecalBlendDesc` / `FVisibleDecal` / ステージ定義
- [[b_dbuffer]] — DBuffer 描画フロー詳細
