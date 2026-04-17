# ref: DeferredLight シェーダークラス群

- 対象: `LightRendering.h` / `LightRendering.cpp`
- Details: [[a_light_rendering]] | [[b_clustered_tiled]]

---

## FDeferredLightUniformStruct（ライト UniformBuffer）

```cpp
// LightRendering.h:23
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FDeferredLightUniformStruct,)
    // シャドウマップ チャンネルマスク（静的シャドウ用）
    SHADER_PARAMETER(FVector4f, ShadowMapChannelMask)
    // ライストフェードアウト: MAD 形式の距離フェードパラメータ
    SHADER_PARAMETER(FVector2f, DistanceFadeMAD)
    // コンタクトシャドウ
    SHADER_PARAMETER(float, ContactShadowLength)
    SHADER_PARAMETER(float, ContactShadowCastingIntensity)
    SHADER_PARAMETER(float, ContactShadowNonCastingIntensity)
    // ボリュメトリックフォグへの散乱強度
    SHADER_PARAMETER(float, VolumetricScatteringIntensity)
    // シャドウ有無フラグ（ビットマスク）
    SHADER_PARAMETER(uint32, ShadowedBits)
    // ライトチャンネルマスク
    SHADER_PARAMETER(uint32, LightingChannelMask)
    // ライスト本体パラメータ（位置・方向・色・減衰等）
    SHADER_PARAMETER_STRUCT_INCLUDE(FLightShaderParameters, LightParameters)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

---

## FDeferredLightVS（ライストバウンダリ頂点シェーダー）

```cpp
// LightRendering.h:299
class FDeferredLightVS : public FGlobalShader
{
    DECLARE_SHADER_TYPE(FDeferredLightVS, Global);
    SHADER_USE_PARAMETER_STRUCT(FDeferredLightVS, FGlobalShader);

    // ラジアルライスト（Point/Spot）か否かのパーミュテーション
    class FRadialLight : SHADER_PERMUTATION_BOOL("SHADER_RADIAL_LIGHT");
    using FPermutationDomain = TShaderPermutationDomain<FRadialLight>;

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
        SHADER_PARAMETER_STRUCT_INCLUDE(FDrawFullScreenRectangleParameters, FullScreenRect)
        SHADER_PARAMETER_STRUCT_INCLUDE(FStencilingGeometryShaderParameters::FParameters, Geometry)
    END_SHADER_PARAMETER_STRUCT()

public:
    // フルスクリーン（Directional）またはバウンダリジオメトリ（Radial）を頂点変換
    static FParameters GetParameters(const FViewInfo& View, bool bBindViewUniform = true);
    static FParameters GetParameters(const FViewInfo& View, const FSphere& LightBounds, bool bBindViewUniform = true);
    static FParameters GetParameters(const FViewInfo& View, const FLightSceneInfo* LightSceneInfo, bool bBindViewUniform = true);
};
```

---

## ELightOcclusionType（シャドウ方式列挙）

```cpp
// LightRendering.h:341
enum class ELightOcclusionType : uint8
{
    Shadowmap,       // 通常シャドウマップ（CSM / Stationary）
    Raytraced,       // HW Ray Tracing（RHI_RAYTRACING が必要）
    MegaLights,      // MegaLights + HW RT シャドウ
    MegaLightsVSM,   // MegaLights + VSM シャドウ
};

// ライストごとの方式取得
ELightOcclusionType GetLightOcclusionType(const FLightSceneProxy& Proxy, const FSceneViewFamily& ViewFamily);
ELightOcclusionType GetLightOcclusionType(const FLightSceneInfoCompact& LightInfo, const FSceneViewFamily& ViewFamily);
```

---

## 主要関数一覧

| 関数 | ファイル | 役割 |
|-----|---------|------|
| `GetDeferredLightParameters()` | `LightRendering.h:40` | `FDeferredLightUniformStruct` 構築 |
| `SetDeferredLightParameters()` | `LightRendering.h:42` | シェーダーへ UniformBuffer バインド |
| `GetShadowedBits()` | `LightRendering.h:39` | ShadowedBits ビットマスク計算 |
| `RenderLight()` | `LightRendering.cpp` | 単一ライストの Deferred シェーディング |
| `RenderLights()` | `LightRendering.cpp:1520` | 全ライスト処理エントリポイント |
| `GetLightFadeFactor()` | `LightRendering.h:37` | 距離フェード係数計算 |

---

> [!note]- FLightShaderParameters の内容
> `FLightShaderParameters` は `LightData.ush` で定義される HLSL 互換構造体。
> 位置（`WorldPosition`）・方向（`Direction`）・色（`Color`）・インバース半径（`InvRadius`）・
> フォールオフ指数（`FalloffExponent`）・スポットアングル cos 値等を含む。
> `FDeferredLightUniformStruct` に `SHADER_PARAMETER_STRUCT_INCLUDE` で埋め込まれる。

> [!note]- ShadowedBits の構造
> `ShadowedBits` は `GetShadowedBits()` で計算されるビットマスク。
> bit 0: 動的シャドウ有無、bit 1: 透明シャドウ有無、bit 2: コンタクトシャドウ有無。
> シェーダーで `(ShadowedBits & 1)` のようにマスクして分岐する。

> [!note]- DistanceFadeMAD の計算
> `DistanceFadeMAD = (FadeDistance, FadeBias)` で `Fade = saturate(Distance * DistanceFadeMAD.x + DistanceFadeMAD.y)`。
> ライスト半径の外縁付近でライスト寄与をソフトにフェードアウトさせる。
