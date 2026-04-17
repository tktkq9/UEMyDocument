# ref: ライストパラメータ構造体

- 対象: `LightRendering.h` / `LightSceneInfo.h` / `SceneManagement.h`
- Details: [[a_light_rendering]] | [[d_light_functions]]

---

## FSortedLightSetSceneInfo（ライスト分類構造体）

```cpp
// LightRendering.cpp（RenderLights で使用）
struct FSortedLightSetSceneInfo
{
    // ライストソート済みリスト（バッチ処理の境界インデックスを保持）
    TArray<FSortedLightSceneInfo, SceneRenderingAllocator> SortedLights;

    // ---- 境界インデックス（値は SortedLights のインデックス） ----
    int32 SimpleLightsEnd;           // Simple Lights の終端（シャドウなし・小規模）
    int32 ClusteredSupportedEnd;     // Clustered Deferred 対応ライスト終端
    int32 UnbatchedLightStart;       // 個別処理開始インデックス
    int32 MegaLightsLightStart;      // MegaLights 管理ライスト開始インデックス

    // Rect Light が存在するか
    bool bHasRectLights;

    // VSM Shadow Mask テクスチャ（ShadowSceneRenderer から渡される）
    FRDGTextureRef VirtualShadowMapMaskBits;
    FRDGTextureRef VirtualShadowMapMaskBitsHairStrands;
};
```

---

## FSortedLightSceneInfo（ソートキー付きライスト情報）

```cpp
// LightRendering.cpp（内部定義）
struct FSortedLightSceneInfo
{
    // ソートキー（シャドウ有無・ライストタイプ・距離等を詰めたビット列）
    union
    {
        struct
        {
            uint32 LightType       : 2;  // ELightComponentType（Point/Spot/Directional/Rect）
            uint32 bTextureProfile : 1;  // IES プロファイル使用
            uint32 bShadowed       : 1;  // 動的シャドウあり
            uint32 bLightFunction  : 1;  // Light Function マテリアル使用
            uint32 bUsesLightingChannels : 1;
            // ...
        } Fields;
        int32 Packed;
    } SortKey;

    const FLightSceneInfo* LightSceneInfo;
    int32 SimpleLightIndex;  // SimpleLight の場合のインデックス（-1 = 通常）
};
```

---

## FLightSceneProxy（ライストシーンプロキシ）

```cpp
// SceneManagement.h（概略）
class FLightSceneProxy
{
public:
    // ---- 基本プロパティ ----
    FVector   Position;           // ワールド空間位置
    FVector   Direction;          // 方向（Directional/Spot）
    FLinearColor Color;           // HDR 色（強度込み）
    float     Radius;             // 影響半径
    float     FalloffExponent;    // 減衰指数（inverse-square=0）
    ELightComponentType LightType;// Point/Spot/Directional/Rect

    // ---- シャドウ設定 ----
    bool      bCastDynamicShadow;
    bool      bCastStaticShadow;
    uint8     ShadowMapChannel;   // Stationary Light の Static Shadow チャンネル（0–3）
    float     ContactShadowLength;

    // ---- ライストチャンネル ----
    uint8     LightingChannelMask;// ビットマスク（0=全オブジェクト）

    // ---- Light Function ----
    const UMaterialInterface* LightFunctionMaterial;

    // ---- MegaLights 設定 ----
    TEnumAsByte<EMegaLightsShadowMethod::Type> MegaLightsShadowMethod;
    bool bAllowMegaLights;
};
```

---

## FLightSceneInfo（シーン内管理構造体）

```cpp
// LightSceneInfo.h（概略）
class FLightSceneInfo : public FRenderResource
{
public:
    FLightSceneProxy*  Proxy;         // レンダープロキシ
    int32              Id;            // FScene::Lights 内インデックス
    bool               bPrecomputedLightingIsValid;

    // Static Shadow チャンネル（0–3, Stationary Light のみ）
    int32              PreviewShadowMapChannel;

    // ライストグリッドへの登録（Forward / Clustered Deferred で参照）
    bool               bIsInForwardLightGrid;
};
```

---

## 主要関数（ライスト分類）

| 関数 | ファイル | 役割 |
|-----|---------|------|
| `GatherLightsAndComputeLightGrid()` | `LightRendering.cpp` | ライスト収集・グリッド構築 |
| `SortAndGroupLights()` | `LightRendering.cpp` | `FSortedLightSetSceneInfo` 構築 |
| `RenderLights()` | `LightRendering.cpp:1520` | ライスト処理ディスパッチ |
| `GetLightOcclusionType()` | `LightRendering.h:349` | シャドウ方式（Shadowmap/RT/MegaLights）判定 |

---

> [!note]- SortKey の意味
> `FSortedLightSceneInfo.SortKey` は Deferred Lighting の処理コストを最小化するために
> シャドウなし→シャドウあり、Directional→Point→Spot→Rect の順にソートされる。
> バッチ境界（SimpleLightsEnd 等）はこのソート後のインデックスを指す。

> [!note]- Static Shadow チャンネル（ShadowMapChannel）
> Stationary Light は最大4個まで存在でき、それぞれ ch 0–3 に割り当てられる。
> `GBufferE` の RGBA 各チャンネルに1ライスト分のプリシャドウファクターが格納される。
> `FDeferredLightUniformStruct.ShadowMapChannelMask` はこのマスクを指す。

> [!note]- FSimpleLightEntry
> パーティクルエミッタ等が生成する小規模な動的ライストは `FSimpleLightEntry` として
> 管理され、 `SimpleLightsEnd` 以前に一括処理される（シャドウなし・Clustered のみ）。
