# ref: FProjectedShadowInfo / FShadowCascadeSettings

- 対象ファイル: `ShadowRendering.h` / `ShadowSetup.cpp`
- 概要: [[17_shadow_overview]]

---

## FProjectedShadowInfo（ShadowRendering.h:278）

Shadow Depth 描画から Projection まで一貫して使われるシャドウの主要構造体。  
`FRefCountedObject` を継承し、参照カウントで管理される。

```cpp
class FProjectedShadowInfo : public FRefCountedObject
{
public:
    // ─── ビュー / レンダーターゲット ──────────────────────────────
    FViewInfo* ShadowDepthView;          // Shadow Depth 描画用ビュー（ライスト視点）
                                         // NOTE: 主ビューを流用してオーバーライドしているため
                                         //       GetShadowDepthRenderingViewMatrices() を使用すること
    FShadowMapRenderTargets RenderTargets; // Shadow Depth Atlas テクスチャ
    EShadowDepthCacheMode CacheMode;       // SDCM_Uncached / StaticPrimitivesOnly / MovablePrimitivesOnly
    FViewInfo* DependentView;              // 依存するメインビュー（NULL = ビュー非依存）
    int32 ShadowId;                        // FVisibleLightInfo::AllProjectedShadows 内インデックス

    // ─── 変換行列 ────────────────────────────────────────────────
    FVector PreShadowTranslation;          // ワールド空間への前変換（精度維持）
    FMatrix TranslatedWorldToView;         // ライスト View 行列
    FMatrix ViewToClipInner;               // View → Clip（ボーダーなし）
    FMatrix ViewToClipOuter;               // View → Clip（ボーダーあり）
    FMatrix44f TranslatedWorldToClipInnerMatrix; // Shadow Depth 描画用 WorldToClip
    FMatrix44f TranslatedWorldToClipOuterMatrix;
    FMatrix44f InvReceiverInnerMatrix;     // 受光面の逆行列

    // ─── 深度パラメータ ──────────────────────────────────────────
    float InvMaxSubjectDepth;              // 深度正規化係数 (= 1.0 / (MaxZ - MinZ))
    float MaxSubjectZ;                     // 遮蔽物の最大深度（ワールド単位）
    float MinSubjectZ;                     // 遮蔽物の最小深度（ワールド単位）
    float MinPreSubjectZ;

    // ─── フラスタム / バウンド ───────────────────────────────────
    FConvexVolume CasterOuterFrustum;      // 遮蔽物含むフラスタム（ボーダーあり）
    FConvexVolume ReceiverInnerFrustum;    // 受光面フラスタム（ボーダーなし）
    FSphere ShadowBounds;                  // シャドウ影響球（カリング用）

    // ─── アトラス座標 ────────────────────────────────────────────
    uint32 X, Y;                           // アトラス内オフセット
    uint32 ResolutionX, ResolutionY;       // Shadow Map サイズ（ボーダーなし）
    uint32 BorderSize;                     // PCF フィルタリング用ボーダー幅
    FIntRect ScissorRectOptim;             // CSM フラスタム外スキップ用シザー

    // ─── カスケード / スケール ───────────────────────────────────
    float MaxScreenPercent;                // ビューに占める最大スクリーン割合
    TArray<float, TInlineAllocator<2>> FadeAlphas; // ビューごとのフェードアルファ

    // ─── フラグ（bitfield）──────────────────────────────────────
    uint32 bAllocated : 1;                 // アトラスに割り当て済み
    uint32 bRendered : 1;                  // Projection 描画済み
    uint32 bAllocatedInPreshadowCache : 1; // Preshadow キャッシュに格納
    uint32 bDepthsCached : 1;              // 深度がキャッシュ済み
    uint32 bDirectionalLight : 1;          // Directional Light かどうか
    uint32 bOnePassPointLightShadow : 1;   // キューブマップ1パス描画
    uint32 bVSM : 1;                       // Virtual Shadow Map かどうか
    uint32 bWholeSceneShadow : 1;          // 全シーンシャドウ（CSM 等）
    uint32 bTranslucentShadow : 1;         // 半透明シャドウ
    uint32 bRayTracedDistanceField : 1;    // Ray Traced Distance Field Shadow
    uint32 bCapsuleShadow : 1;             // カプセルシャドウ
    uint32 bPreShadow : 1;                 // PreShadow（静的環境 → 動的受光）
    uint32 bSelfShadowOnly : 1;            // セルフシャドウのみ（FPS 武器向け）
    uint32 bPerObjectOpaqueShadow : 1;     // Per-Object シャドウ
    uint32 bTransmission : 1;              // バックライト透過
    uint32 bHairStrandsDeepShadow : 1;     // Hair Strands ディープシャドウ
    uint32 bNaniteGeometry : 1;            // Nanite ジオメトリ描画

    // ─── カスケード設定 ─────────────────────────────────────────
    FShadowCascadeSettings CascadeSettings;
};
```

---

## FShadowCascadeSettings

CSM カスケードの分割と遷移を制御するパラメータ群。

```cpp
// LightSceneInfo.h（UE5 では SceneManagement.h に定義）
struct FShadowCascadeSettings
{
    // ─── カスケード分割平面 ──────────────────────────────────────
    FPlane ShadowSplitPlane;               // このカスケードの遠端分割平面
    FPlane NearFadePlane;                  // フェード開始平面（前のカスケードとの境界）
    FPlane FarFadePlane;                   // フェード終了平面（次のカスケードとの境界）
    FPlane ShadowSplitPlaneOuter;          // 外側（ボーダーあり）の分割平面

    // ─── 分割距離（ビュー空間）────────────────────────────────
    float SplitNear;                       // このカスケードの近端距離
    float SplitFar;                        // このカスケードの遠端距離
    float SplitNearFadeRegion;             // Near フェード幅
    float SplitFarFadeRegion;              // Far フェード幅（隣カスケードとのブレンド幅）
    float FadePlaneOffset;                 // フェード平面オフセット
    float FadePlaneLength;                 // フェード長さ

    // ─── カスケードインデックス ──────────────────────────────────
    int32 ShadowSplitIndex;                // このカスケードのインデックス（0-3）

    // ─── カリング制御 ────────────────────────────────────────────
    bool bFarShadowCascade;               // 遠距離カスケードかどうか
};
```

---

## EShadowDepthCacheMode

```cpp
// ShadowRendering.h
enum EShadowDepthCacheMode
{
    SDCM_Uncached                  = 0, // キャッシュなし（毎フレーム再描画）
    SDCM_MovablePrimitivesOnly     = 1, // 動的プリミティブのみ
    SDCM_StaticPrimitivesOnly      = 2, // 静的プリミティブのみ（キャッシュ）
};
```

---

## 主要メソッド

| メソッド | ファイル | 説明 |
|---------|---------|------|
| `SetupWholeSceneProjection()` | ShadowSetup.cpp | CSM カスケードの ViewProj 行列構築 |
| `SetupSpotProjection()` | ShadowSetup.cpp | Spot Light の錐台投影設定 |
| `SetupCubemapProjection()` | ShadowSetup.cpp | Point Light の 6 面キューブ設定 |
| `RenderDepth()` | ShadowDepthRendering.cpp | Shadow Depth Map 描画呼び出し |
| `RenderProjectedShadow()` | ShadowRendering.cpp | Shadow Mask 生成（PCF/PCSS）|
| `ComputeScissorRectOptim()` | ShadowRendering.cpp | CSM フラスタム最適化シザー計算 |
| `GetShadowDepthRenderingViewMatrices()` | ShadowRendering.h | 正確な Shadow ViewProj を返す |

> **補足**: `TranslatedWorldToClipInnerMatrix` は頂点シェーダーで使用するライスト視点の  
> WorldToClip 行列。CSM では近平面フラット化（頂点をニアプレーンに押し付け）処理を含む。

---

## 関連リファレンス

- [[ref_shadow_rendering]] — `FShadowDepthVS` / `FShadowDepthMeshProcessor`
- [[ref_shadow_projection]] — `TShadowProjectionPS` / PCF サンプルパターン
- [[a_shadow_setup]] — `FProjectedShadowInfo` 生成フロー
