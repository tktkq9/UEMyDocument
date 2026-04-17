# ref: FHZBParameters / EHZBType / HZB テクスチャ仕様

- 対象ファイル: `HZB.h/.cpp`
- 概要: [[21_hzb_overview]]

---

## EHZBType（HZB.h:44）

```cpp
enum class EHZBType : uint8
{
    Dummy       = 0,             // ダミー（HZB 無効時のフォールバック）
    ClosestHZB  = 1,             // 最小深度（最も近い）
    FurthestHZB = 2,             // 最大深度（最も遠い）
    All         = ClosestHZB | FurthestHZB,
};
```

---

## FHZBParameters（HZB.h:14）

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FHZBParameters, )
    // ─── デフォルト HZB（FurthestHZB）────────────────────────────
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, HZBTexture)
    SHADER_PARAMETER_SAMPLER(SamplerState, HZBSampler)
    SHADER_PARAMETER_SAMPLER(SamplerState, HZBTextureSampler)

    // ─── ClosestHZB（最小深度）──────────────────────────────────
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ClosestHZBTexture)
    SHADER_PARAMETER_SAMPLER(SamplerState, ClosestHZBTextureSampler)

    // ─── FurthestHZB（最大深度）─────────────────────────────────
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, FurthestHZBTexture)
    SHADER_PARAMETER_SAMPLER(SamplerState, FurthestHZBTextureSampler)

    // ─── 共通パラメータ ──────────────────────────────────────────
    SHADER_PARAMETER(FVector2f, HZBSize)               // HZB テクスチャ全体サイズ（px）
    SHADER_PARAMETER(FVector2f, HZBViewSize)           // ビュー内の有効サイズ
    SHADER_PARAMETER(FIntRect,  HZBViewRect)           // ビューポート矩形
    SHADER_PARAMETER(FVector2f, ViewportUVToHZBBufferUV) // ビューポート UV → HZB UV 変換
    SHADER_PARAMETER(FVector4f, HZBUvFactorAndInvFactor)
    SHADER_PARAMETER(FVector4f, HZBUVToScreenUVScaleBias)
    SHADER_PARAMETER(FVector2f, HZBBaseTexelSize)       // Mip0 のテクセルサイズ
    SHADER_PARAMETER(FVector2f, SamplePixelToHZBUV)    // スクリーンピクセル → HZB UV
    SHADER_PARAMETER(uint32,    bIsHZBValid)
    SHADER_PARAMETER(uint32,    bIsFurthestHZBValid)
    SHADER_PARAMETER(uint32,    bIsClosestHZBValid)
    SHADER_PARAMETER(FVector4f, ScreenPosToHZBUVScaleBias)
    SHADER_PARAMETER(FVector2f, DummyHZB)
END_SHADER_PARAMETER_STRUCT()
```

---

## HZB テクスチャ仕様

| 項目 | FurthestHZB | ClosestHZB |
|------|-------------|------------|
| フォーマット | `PF_R16F` | `PF_R16F` |
| 解像度 | `NextPOT(ViewWidth/2) × NextPOT(ViewHeight/2)` | 同左 |
| ミップ数 | `FloorLog2(Max(W,H)) + 1` | 同左 |
| Min/Max | 各 2×2 の **最小値**（= 最も遠い深度）| 各 2×2 の **最大値**（= 最も近い深度）|
| サンプラー | Point Clamp | Point Clamp |

> **Reverse-Z 注意**: UE5 は Reverse-Z（遠方 = 0.0, 近傍 = 1.0）。  
> FurthestHZB は「最も遠い」＝ 深度値の最小値を保持する。

---

## GetHZBParameters オーバーロード（HZB.h）

```cpp
// 明示的にテクスチャを指定する版
FHZBParameters GetHZBParameters(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    EHZBType InHZBTypes,
    FRDGTextureRef InClosestHZB,
    FRDGTextureRef InFurthestHZB);

// View から現在フレームの HZB を参照する版
FHZBParameters GetHZBParameters(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    EHZBType InHZBTypes);

// 前フレームの HZB へのフォールバックを許可する版
// （Nanite Persistent Cull などで使用）
FHZBParameters GetHZBParameters(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    bool bUsePreviousHZBAsFallback);

// ダミー（HZB 無効時のフォールバック）
FHZBParameters GetDummyHZBParameters(FRDGBuilder& GraphBuilder);
```

---

## IsHZBValid チェック

```cpp
// HZB が構築済みかどうか確認（利用前に必ずチェック）
bool IsHZBValid(
    const FViewInfo& View,
    EHZBType InHZBTypes,
    bool bCheckIfProduced = false); // true: 今フレーム生成済みかも確認

bool IsPreviousHZBValid(const FViewInfo& View, EHZBType InHZBTypes);

// テクスチャを直接取得
FRDGTextureRef GetHZBTexture(const FViewInfo& View, EHZBType InHZBTypes);
```

---

## HZB を使うシステムと参照フィールド

| システム | 参照フィールド | EHZBType |
|---------|-------------|---------|
| SSR | `FViewInfo::HZB` | FurthestHZB |
| Lumen Screen Probe | `GetHZBParameters(View, FurthestHZB)` | FurthestHZB |
| MegaLights | `FMegaLightsParameters::HZBParameters` | All |
| Nanite Persistent Cull | `GetHZBParameters(View, true)` | FurthestHZB |
| HZB Occlusion | `RenderOcclusion(View.HZB)` | FurthestHZB |
| Contact Shadow | `ClosestHZBTexture` | ClosestHZB |

---

## 関連リファレンス

- [[ref_hzb_occlusion]] — `FViewOcclusionQueries` / `FOcclusionQueryVS`
- [[a_hzb_build]] — BuildHZB() 構築フロー
