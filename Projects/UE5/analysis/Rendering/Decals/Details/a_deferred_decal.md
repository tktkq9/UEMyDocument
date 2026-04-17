# a: Deferred Decal 描画（RenderDeferredDecals）

- 対象ファイル: `DecalRenderingShared.cpp` / `DecalRenderingCommon.h`
- 概要: [[15_decals_overview]]

---

## 概要

`RenderDeferredDecals()` は GBuffer デカール（BasePass 後のライティング前）を描画し、  
深度テスト + ステンシルによって投影ボックス（OBB）の範囲に限定して  
GBuffer チャンネルを上書きする。

---

## EDecalRenderStage と EDecalRenderTargetMode の対応

```
EDecalRenderStage            EDecalRenderTargetMode
────────────────────────────────────────────────────
BeforeBasePass            →  DBuffer
BeforeLighting (Stain)    →  SceneColorAndGBuffer
BeforeLighting (Normal)   →  SceneColorAndGBufferNoNormal
Emissive                  →  SceneColor
AmbientOcclusion          →  AmbientOcclusion
```

---

## FDecalBlendDesc（FDecalRenderingCommon.h）

```cpp
union FDecalBlendDesc
{
    uint32 Packed = 0;
    struct
    {
        uint32 BlendMode                    : 8;  // マテリアルの Decal Blend Mode
        uint32 RenderStageMask              : 8;  // 対応するステージのビットマスク
        uint32 bWriteBaseColor              : 1;  // BaseColor チャンネルに書くか
        uint32 bWriteNormal                 : 1;  // Normal チャンネルに書くか
        uint32 bWriteRoughnessSpecularMetallic : 1;
        uint32 bWriteEmissive               : 1;
        uint32 bWriteAmbientOcclusion       : 1;
        uint32 bWriteDBufferMask            : 1;  // DBuffer マスクに書くか
    };
};
// → DecalRendering::ComputeDecalBlendDesc() でマテリアルから計算
```

---

## FVisibleDecal（DecalRenderingShared.h）

```cpp
struct FVisibleDecal
{
    const FMaterialRenderProxy* MaterialProxy; // デカールマテリアル
    uintptr_t Component;                       // UDecalComponent へのポインタ
    uint32 SortOrder;                          // 描画順序
    FDecalBlendDesc BlendDesc;                 // ブレンドモード情報
    float ConservativeRadius;                  // 投影球の保守半径
    float FadeAlpha;                           // フェードアルファ
    float InvFadeDuration;                     // フェードアウト速度
    float InvFadeInDuration;
    float FadeStartDelayNormalized;
    float FadeInStartDelayNormalized;
    FLinearColor DecalColor;                   // デカール色（Tint）
    FTransform ComponentTrans;                 // デカールコンポーネント Transform
    FBox BoxBounds;                            // ワールド空間の AABB
};
```

---

## RenderDeferredDecals() フロー（GBuffer Decals）

```
RenderDeferredDecals(GraphBuilder, Views, SceneTextures, Stage)
  │
  ├─ [A] 可視デカールの取得
  │   FDecalVisibilityTaskData::FinishRelevantDecals(Stage)
  │   → InitViews 時に非同期で計算済みのデカールリストを取得
  │   → FrustumCull + DistanceCull で不可視デカールを除外
  │
  ├─ [B] ステージ別 RT バインド
  │   Stage == BeforeLighting:
  │     RT0 = SceneColor, RT1-4 = GBufferA/B/C/E
  │   Stage == AmbientOcclusion:
  │     RT0 = SSAOTexture
  │
  ├─ [C] デカールをソートして描画
  │   for each FVisibleDecal in RelevantDecals（SortOrder 昇順）:
  │     │
  │     ├─ [C1] ステンシルマスク設定
  │     │   → 投影 OBB の正面/背面判定に基づいてステンシル書き込み
  │     │   → r.Decal.StencilBackFaceSelection=1 の場合:
  │     │      背面のみステンシルを立て、フロントフェースで描画
  │     │      （内側にカメラがある場合の対策）
  │     │
  │     ├─ [C2] シェーダーバインド
  │     │   FDeferredDecalPS（GetDecalShaders() で取得）
  │     │   → SetParameters(): DecalBlendDesc / ComponentTrans / FadeAlpha
  │     │   → ShadowMap テクスチャが必要な場合はバインド
  │     │
  │     └─ [C3] 投影ボックスを DrawCall
  │             DrawIndexedPrimitive(UnitBox) でデカール OBB を描画
  │             → VS で ComponentTrans × UnitBox → WorldPos
  │             → PS で WorldPos → GBuffer チャンネルを上書き
  │
  └─ [D] ステンシルクリア
      デカール描画後にステンシルをリセット
```

---

## デカール投影の仕組み

```
PS 内での処理（DecalShader.usf）:

1. SV_Position → スクリーン UV → SceneDepth から WorldPos を再構築
2. WorldPos を DecalOBB のローカル空間に変換:
   DecalUV = (WorldToDecal × WorldPos).xy + 0.5
   (x, y ∈ [0, 1] の範囲外なら discard)
3. MaterialTexture.Sample(DecalUV) → BaseColor / Normal / RM を取得
4. FadeAlpha で強度を減衰
5. BlendDesc に応じて GBuffer チャンネルを上書き

法線のブレンド:
  Stain/Translucent: Normal = lerp(GBufferNormal, DecalNormal, Opacity)
  Normal Only:       Normal = DecalNormal（既存法線を完全上書き）
```

---

## 関連リファレンス

- [[ref_deferred_decal]] — `FDeferredDecalPS` / `FDecalBlendDesc` 詳細
- [[b_dbuffer]] — DBuffer デカール（BasePass 前の事前合成）
