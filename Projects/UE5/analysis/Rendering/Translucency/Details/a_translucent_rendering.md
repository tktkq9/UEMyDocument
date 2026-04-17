# a: Translucency 描画（RenderTranslucency）

- 対象ファイル: `TranslucentRendering.h/.cpp` / `BasePassRendering.cpp`
- 概要: [[20_translucency_overview]]

---

## 概要

`RenderTranslucency()` は **半透明マテリアルを持つオブジェクトを  
深度ソート順に Back→Front で描画するパス**。  
GBuffer が使えないため Forward Shading で照明計算を行い、  
`TranslucencyVolume` から間接光を取得する。

---

## ETranslucencyPass（パス種別）

```cpp
// TranslucentRendering.h
namespace ETranslucencyPass
{
    enum Type : int
    {
        TPT_StandardTranslucency,         // Before DOF（通常の半透明）
        TPT_TranslucencyAfterDOF,         // After DOF（Separate Translucency）
        TPT_TranslucencyAfterDOFModulate, // After DOF の乗算合成
        TPT_TranslucencyAfterMotionBlur,  // Motion Blur 後
        TPT_AllTranslucency,              // 全種別（分類用）
        TPT_MAX
    };
}
```

---

## Separate Translucency（DOF との分離）

```
r.SeparateTranslucency=1（デフォルト）の場合:

マテリアルの "Separate Translucency" フラグ（マテリアルエディタで設定）
  └─ true:  TPT_TranslucencyAfterDOF パスに回す（DOF の影響を受けない）
     false: TPT_StandardTranslucency パスで描画（DOF の影響を受ける）

Separate Translucency は低解像度 RT（r.SeparateTranslucencyScreenPercentage）に
描画してから SceneColor に合成することで、DOF 後のノイズを軽減する。
```

---

## 深度ソート

```cpp
// TranslucentRendering.cpp
// 半透明オブジェクトはカメラ距離でソート（遠い順）
// Back-to-Front ソートでアルファブレンドの正確な重ね合わせを実現

FTranslucentPrimSet::SortPrimitives()
    → View.ViewPoint とプリミティブの BoundsOrigin の距離でソート
    → 距離降順（最も遠いものから描画）

// 注意: メッシュ内部のトライアングルは未ソート
// → 正確なピクセル単位の透過順は OIT が必要
```

---

## FTranslucentBasePassMeshProcessor

```cpp
// BasePassRendering.cpp（半透明用 MeshPassProcessor）
// 内部的には FBasePassMeshProcessor と同じシェーダーを使用するが
// ブレンドステートとライティングパスが異なる

// ブレンドステート:
//   Translucent: SrcColor + (1-SrcAlpha) × DstColor
//   Additive:    SrcColor + DstColor
//   Modulate:    SrcColor × DstColor

// ライティング:
//   FOpaqueBasePassUniformParameters の代わりに
//   FTranslucentBasePassUniformParameters を使用
//   → SceneTextures（GBuffer）を参照可能
//   → TranslucencyVolume テクスチャも含む
```

---

## RenderTranslucency() フロー

```
RenderTranslucency()                            TranslucentRendering.cpp
  │
  ├─ [A] 前処理
  │   CopySceneColorForTranslucency()
  │   → 「屈折」マテリアルが前フレームの SceneColor を参照するためコピー
  │
  ├─ [B] TPT_StandardTranslucency パス
  │   RenderTranslucencyInner(..., TPT_StandardTranslucency)
  │     │
  │     ├─ SceneColor RT にバインド（ELoad → 既存の不透明描画に上書き）
  │     │
  │     ├─ for each Primitive in SortedTranslucentPrimitives（遠い順）:
  │     │   FTranslucentBasePassMeshProcessor::AddMeshBatch()
  │     │     → FTranslucentBasePassUniformParameters バインド
  │     │       （TranslucencyVolume / SceneTextures / ReflectionCapture 等）
  │     │     → Forward Shading:
  │     │         GetForwardLightData() からライスト一覧取得
  │     │         GGX + Lambertian BRDF 評価
  │     │         TranslucencyVolume.SampleLevel(WorldPos) → 間接光
  │     │     → アルファブレンドで SceneColor に加算
  │     │
  │     └─ [OIT 有効時] AccumulateOIT 代わりに OIT RT に書き込み（後で合成）
  │
  ├─ [C] DOF / Motion Blur 処理（この間）
  │
  ├─ [D] TPT_TranslucencyAfterDOF パス
  │   RenderTranslucencyInner(..., TPT_TranslucencyAfterDOF)
  │     → Separate Translucency RT（低解像度可）に描画
  │     → UpscaleTranslucencyIfNeeded() でアップスケール
  │     → CompositeTranslucentSceneColor() で SceneColor に合成
  │
  └─ [E] OIT Composite（OIT 有効時）
      CompositeOIT()
      → AccumRT（加重合計色）/ RevealageRT（1 - 加重アルファ）から
        最終色を計算して SceneColor に書き込み

【深度ソート → 描画 詳細フロー】
  FTranslucentPrimSet::DrawPrimitives(View, DrawRenderState, ...):
    for each VisiblePrimitive in DistanceSorted:
      Primitive->Proxy->GetDynamicMeshElements()
        → FMeshBatch を収集
      FTranslucentBasePassMeshProcessor::AddMeshBatch(MeshBatch)
        → GetUniformLightMapPolicy() でポリシー確定
        → TBasePassPS<LightMapPolicy> + ブレンドステート設定
        → FMeshDrawCommand 生成・Submit
```

---

## 関連リファレンス

- [[ref_translucent_rendering]] — `FTranslucencyPassResources` / `FTranslucentPrimSet`
- [[b_translucent_lighting]] — TranslucencyVolume 照明注入
- [[c_oit]] — Order-Independent Translucency
