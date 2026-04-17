# ref: FTranslucencyPassResources / FTranslucentPrimSet / FOITData

- 対象ファイル: `TranslucentRendering.h/.cpp` / `OIT/OIT.h`
- 概要: [[20_translucency_overview]]

---

## ETranslucencyPass::Type（TranslucentRendering.h）

```cpp
namespace ETranslucencyPass
{
    enum Type : int
    {
        TPT_StandardTranslucency,          // Before DOF（通常の半透明）
        TPT_TranslucencyAfterDOF,          // After DOF（Separate Translucency）
        TPT_TranslucencyAfterDOFModulate,  // After DOF の乗算合成
        TPT_TranslucencyAfterMotionBlur,   // Motion Blur 後
        TPT_AllTranslucency,               // 全種別（ループ用）
        TPT_MAX
    };
}
```

---

## FTranslucentBasePassUniformParameters

```cpp
// BasePassRendering.h（半透明用 UniformBuffer）
BEGIN_UNIFORM_BUFFER_STRUCT(FTranslucentBasePassUniformParameters, )
    // FOpaqueBasePassUniformParameters と同じフィールド +
    UNIFORM_MEMBER_STRUCT(FSceneTextureUniformParameters, SceneTextures)
        // → SceneDepthTexture, GBufferA-F にアクセス可能
        //   （屈折・シーン法線参照に必要）
    UNIFORM_MEMBER_TEXTURE(Texture3D, TranslucencyLightingVolumeAmbientInner)
    UNIFORM_MEMBER_TEXTURE(Texture3D, TranslucencyLightingVolumeAmbientOuter)
    UNIFORM_MEMBER_TEXTURE(Texture3D, TranslucencyLightingVolumeDirectionalInner)
    UNIFORM_MEMBER_TEXTURE(Texture3D, TranslucencyLightingVolumeDirectionalOuter)
    // MegaLights TranslucencyVolume（有効時）
    UNIFORM_MEMBER_TEXTURE(Texture3D, MegaLightsTranslucencyVolumeAmbient)
    UNIFORM_MEMBER_TEXTURE(Texture3D, MegaLightsTranslucencyVolumeDirectional)
    // Lumen Radiance Cache（Lumen 有効時）
    UNIFORM_MEMBER_STRUCT(FLumenTranslucencyLightingParameters, LumenParameters)
END_UNIFORM_BUFFER_STRUCT()
```

---

## FTranslucentBasePassMeshProcessor（BasePassRendering.cpp）

```cpp
// 半透明マテリアル専用の MeshPassProcessor
class FTranslucentBasePassMeshProcessor : public FMeshPassProcessor
{
    // ─── コンストラクタパラメータ ───────────────────────────────────
    //   TranslucencyPassType: ETranslucencyPass::Type
    //   InTranslucentBasePassUniformBuffer: UB バインド
    //   InPassDrawRenderState: ブレンドステート / 深度ステート

    virtual void AddMeshBatch(...) override;
    // → マテリアルのブレンドモードに応じてブレンドステートを選択:
    //   Translucent:  SrcColor + (1-SrcAlpha) × DstColor
    //   Additive:     SrcColor + DstColor
    //   Modulate:     SrcColor × DstColor
    //   AlphaComposite: Pre-multiplied Alpha
    //
    // → FTranslucentBasePassUniformParameters をバインド
    // → Forward Shading: GetForwardLightData() からライスト一覧取得
    //   GGX + Lambertian BRDF 評価
    //   TranslucencyVolume.SampleLevel(WorldPos) → 間接光
};
```

---

## 深度ソート（FTranslucentPrimSet）

```cpp
// TranslucentRendering.h
class FTranslucentPrimSet
{
    // 半透明プリミティブの深度ソート済みリスト

    // ソート: カメラ距離降順（遠い順）
    void SortPrimitives()
    {
        // View.ViewPoint とプリミティブ BoundsOrigin の距離を計算
        // std::sort で距離降順ソート
        // ⚠ メッシュ内トライアングルは未ソート → 正確な OIT は別途必要
    }

    TArray<FTranslucentPrimData> SortedPrims;
    // FTranslucentPrimData:
    //   FPrimitiveSceneProxy* Proxy
    //   float SortKey  (= カメラ距離の二乗)
    //   ETranslucencyPass::Type PassType
};
```

---

## Separate Translucency フロー

```
マテリアルエディタ設定: "Separate Translucency" フラグ
  └─ true:  TPT_TranslucencyAfterDOF パスに振り分け
            → 低解像度 RT（r.SeparateTranslucencyScreenPercentage）に描画
            → DOF の影響を受けない（ガラス/霧などの「ぼけてはいけない」素材向け）
     false: TPT_StandardTranslucency パス（DOF の影響を受ける）

r.SeparateTranslucency=0 の場合:
  → 全半透明が TPT_StandardTranslucency パスに統合される
```

---

## FTranslucencyPassResources

```cpp
// TranslucentRendering.h（RDG リソースまとめ）
struct FTranslucencyPassResources
{
    FRDGTextureRef ColorTexture = nullptr;       // 半透明描画先 RT（SceneColor or 別 RT）
    FRDGTextureRef ColorModulateTexture = nullptr; // AfterDOFModulate 用
    FRDGTextureRef DepthTexture = nullptr;        // 半透明 Depth（Separate RT 用）
    ETranslucencyPass::Type Pass = TPT_MAX;

    bool IsValid() const { return ColorTexture != nullptr; }
};
```

---

## CopySceneColorForTranslucency（屈折対応）

```
CopySceneColorForTranslucency()             TranslucentRendering.cpp
  → 屈折マテリアルが SceneColor を参照するために事前コピー
  → FSceneViewState::TemporalSceneColor に保存
  → マテリアル内: SceneTextureLookup(ScreenUV, PreviousSceneColor)

実行条件:
  r.RefractionQuality > 0 かつ 屈折マテリアルが存在する場合
```

---

## UpscaleTranslucencyIfNeeded（Separate RT）

```
Separate Translucency の解像度設定:
  r.SeparateTranslucencyScreenPercentage = 100 → フル解像度
  r.SeparateTranslucencyScreenPercentage = 50  → 半解像度（パフォーマンス向上）

低解像度で描画後:
  UpscaleTranslucencyIfNeeded()
  → FUpscalePS（Bilinear / Temporal）でアップスケール
  CompositeTranslucentSceneColor()
  → SceneColor に最終合成
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.SeparateTranslucency` | 1 | Separate Translucency RT 有効 |
| `r.SeparateTranslucencyScreenPercentage` | 100 | Separate RT の解像度スケール（%）|
| `r.TranslucentSortPolicy` | 0 | 0=カメラ距離 1=プロジェクション座標 |
| `r.RefractionQuality` | 2 | 屈折品質（0=無効）|

---

## 関連リファレンス

- [[ref_translucent_lighting]] — `FTranslucencyLightingVolumeTextures`
- [[a_translucent_rendering]] — RenderTranslucency() 詳細フロー
- [[c_oit]] — OIT（Order-Independent Translucency）
