# ref: FDecalBlendDesc / FVisibleDecal / FDecalVisibilityTaskData

- 対象ファイル: `DecalRenderingCommon.h` / `DecalRenderingShared.h`
- 概要: [[15_decals_overview]]

---

## FDecalBlendDesc（DecalRenderingCommon.h:18）

```cpp
// デカールのブレンドモードを 32bit にパックした記述子
union FDecalBlendDesc
{
    uint32 Packed = 0;
    struct
    {
        uint32 BlendMode                       : 8;  // EDecalBlendMode
        uint32 RenderStageMask                 : 8;  // 有効な EDecalRenderStage のビットマスク
        uint32 bWriteBaseColor                 : 1;
        uint32 bWriteNormal                    : 1;
        uint32 bWriteRoughnessSpecularMetallic : 1;
        uint32 bWriteEmissive                  : 1;
        uint32 bWriteAmbientOcclusion          : 1;
        uint32 bWriteDBufferMask               : 1;  // DBuffer Mask チャンネル書き込み
    };
};
```

---

## EDecalRenderStage（DecalRenderingCommon.h:35）

```cpp
enum class EDecalRenderStage : uint8
{
    None = 0,
    BeforeBasePass       = 1, // DBuffer デカール（GBuffer 前）
    BeforeLighting       = 2, // GBuffer デカール（ライティング前）
    Mobile               = 3,
    MobileBeforeLighting = 4,
    Emissive             = 5, // Emissive のみ書き込み
    AmbientOcclusion     = 6, // AO バッファへの書き込み
    Num,
};
```

---

## EDecalRenderTargetMode（DecalRenderingCommon.h:60）

```cpp
enum class EDecalRenderTargetMode : uint8
{
    None = 0,
    DBuffer                     = 1, // DBufferA/B/C
    SceneColorAndGBuffer        = 2, // SceneColor + GBufferA-E
    SceneColorAndGBufferNoNormal = 3, // Normal を GBuffer から読む場合（法線デカール以外）
    SceneColor                  = 4, // SceneColor のみ（Emissive）
    AmbientOcclusion            = 5, // AO バッファ
    Num,
};
```

---

## DecalRendering 名前空間の主要関数（DecalRenderingCommon.h:86）

```cpp
namespace DecalRendering
{
    // マテリアルから FDecalBlendDesc を計算
    FDecalBlendDesc ComputeDecalBlendDesc(EShaderPlatform, const FMaterial&);

    // デカールが指定ステージで描画すべきかどうか
    bool IsCompatibleWithRenderStage(FDecalBlendDesc, EDecalRenderStage);

    // RT モードとシェーディングパスからメインステージを返す
    EDecalRenderStage GetRenderStage(EDecalRenderTargetMode, EShadingPath);

    // ブレンド記述からメインの描画ステージを返す
    EDecalRenderStage GetBaseRenderStage(FDecalBlendDesc);

    // デカールが指定 RT モードで描画するかどうか
    EDecalRenderTargetMode GetRenderTargetMode(FDecalBlendDesc, EDecalRenderStage);

    // 全 RT モードのビットマスクを返す
    uint8 GetDecalRenderTargetModeMask(const FMaterial&, ERHIFeatureLevel::Type);

    // MeshPassProcessor の種別を返す
    EMeshPass::Type GetMeshPassType(EDecalRenderTargetMode);

    // 描画に必要な RT 数を返す
    uint32 GetRenderTargetCount(FDecalBlendDesc, EDecalRenderTargetMode);

    // RT の書き込みマスクを返す
    uint32 GetRenderTargetWriteMask(FDecalBlendDesc, EDecalRenderStage, EDecalRenderTargetMode);

    // ブレンドステートを返す
    FRHIBlendState* GetDecalBlendState(FDecalBlendDesc, EDecalRenderStage, EDecalRenderTargetMode);
}
```

---

## FVisibleDecal（DecalRenderingShared.h:24）

```cpp
struct FVisibleDecal
{
    const FMaterialRenderProxy* MaterialProxy; // デカールマテリアル
    uintptr_t Component;           // UDecalComponent ポインタ（型消去）
    uint32 SortOrder;              // Sort Priority（マテリアルで設定）
    FDecalBlendDesc BlendDesc;     // ブレンドモード情報
    float ConservativeRadius;      // 投影ボックスの保守的な外接球半径
    float FadeAlpha;               // 現在フレームのフェードアルファ（0〜1）
    float InvFadeDuration;         // フェードアウト速度（= 1 / FadeDuration）
    float InvFadeInDuration;       // フェードイン速度
    float FadeStartDelayNormalized;
    float FadeInStartDelayNormalized;
    FLinearColor DecalColor;       // デカールの色 Tint
    FTransform ComponentTrans;     // デカールコンポーネントの Transform（OBB 変換に使用）
    FBox BoxBounds;                // ワールド空間の AABB（カリング用）
};

using FVisibleDecalList = TArray<FVisibleDecal, SceneRenderingAllocator>;
using FRelevantDecalList = TArray<const FVisibleDecal*, SceneRenderingAllocator>;
```

---

## FDecalVisibilityTaskData（DecalRenderingShared.h:94）

```cpp
class FDecalVisibilityTaskData
{
public:
    // 非同期タスクでデカール可視判定を起動
    static FDecalVisibilityTaskData* Launch(
        FRDGBuilder& GraphBuilder,
        const FScene& Scene,
        TConstArrayView<FViewInfo> Views);

    // 全可視デカールリストを返す（タスク完了を待機）
    TConstArrayView<FVisibleDecal> FinishVisibleDecals(int32 ViewIndex);
};

class FDecalVisibilityViewPacket
{
    // ビューごとの可視判定パケット
    TConstArrayView<FVisibleDecal> FinishVisibleDecals();
    TConstArrayView<const FVisibleDecal*> FinishRelevantDecals(EDecalRenderStage Stage);
    bool HasStage(EDecalRenderStage Stage) const;
    void Finish();
};
```

---

## 関連リファレンス

- [[ref_dbuffer]] — `FDBufferTextures` / `FDBufferData`
- [[a_deferred_decal]] — GBuffer デカール描画フロー
