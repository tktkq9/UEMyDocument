# b: DBuffer デカール（Pre-GBuffer 合成）

- 対象ファイル: `DecalRenderingShared.cpp` / `DecalRenderingCommon.h`
- 概要: [[15_decals_overview]]

---

## 概要

DBuffer デカールは **BasePass 前**に実行されるため、  
静的ライティング（事前計算済みライトマップ）に影響を与えられる。  
GBuffer デカールはライティング後に GBuffer を上書きするだけなのに対して、  
DBuffer デカールは BasePass のシェーダー内で読み取られ正しく合成される。

---

## DBuffer テクスチャ構成

| テクスチャ | フォーマット | 格納内容 |
|-----------|------------|---------|
| DBufferA | R8G8B8A8 | BaseColor(RGB) + Opacity マスク |
| DBufferB | R8G8B8A8 | Normal(RG) + Smoothness オフセット |
| DBufferC | R8G8B8A8 | MetallicSpecularRoughness(RGB) + Opacity マスク |

---

## RenderDBufferDecals() フロー

```
RenderDBufferDecals(GraphBuilder, Views, SceneTextures)
  │
  ├─ [A] DBuffer テクスチャを RDG で確保
  │   DBufferA: R8G8B8A8_UNORM（BaseColor + マスク）
  │   DBufferB: R8G8B8A8_UNORM（Normal + Smoothness）
  │   DBufferC: R8G8B8A8_UNORM（RoughnessMetallicSpecular + マスク）
  │
  ├─ [B] DBuffer クリア（デフォルト値で初期化）
  │   A = (0.5, 0.5, 0.5, 1.0)   // 中立色、完全マスクなし
  │   B = (0.5, 0.5, 0.0, 1.0)   // 中立法線、Smoothness なし
  │   C = (0.0, 0.5, 1.0, 1.0)   // Metallic=0, Roughness=0.5, Spec=1.0
  │
  ├─ [C] EDecalRenderStage::BeforeBasePass デカールを描画
  │   for each FVisibleDecal in BeforeBasePassDecals（SortOrder 昇順）:
  │     ├─ ステンシル設定（a_deferred_decal と同様）
  │     ├─ DBufferA/B/C を RT としてバインド
  │     ├─ FDeferredDecalPS（DBuffer モード）でシェーダー描画
  │     │   → DecalUV を計算して DBufferA/B/C に書き込み
  │     │   → アルファ = デカールの Opacity
  │     └─ 各デカールが加重累積（前のデカールを上塗り）
  │
  └─ [D] DBuffer テクスチャを FSceneTextures に登録
      → BasePass Uniform Buffer に含まれる（FOpaqueDBufferUniformParameters）

BasePass での適用（BasePassPixelShader.usf）:
  FDBufferData DBuffer = GetDBufferData(ScreenUV);
  // DBuffer.PreMulBaseColor / Normal / PremulRoughnessMeallicSpecular
  ApplyDBufferData(DBuffer, BaseColor, Normal, Roughness, Metallic, Specular);
  // → BaseColor = lerp(BaseColor, DBuffer.BaseColor, DBuffer.OpacityMask)
  // → Normal    = lerp(Normal,    DBuffer.Normal,    DBuffer.OpacityMask)
  // → ...
```

---

## GBuffer Decal との比較

| 項目 | GBuffer Decal | DBuffer Decal |
|------|--------------|--------------|
| 実行タイミング | BasePass 後・ライティング前 | BasePass 前 |
| 静的ライティング対応 | 非対応（事前計算に反映されない）| 対応 |
| 半透明サーフェス対応 | 非対応 | 非対応（両方とも）|
| コスト | GBuffer に直接書き込み | 別途 DBuffer RT + BasePass でのサンプル |
| 遮蔽物の深度情報 | SceneDepth を利用 | SceneDepth を利用 |

---

## r.DBuffer の影響

```
r.DBuffer=0 の場合:
  → DBuffer テクスチャを生成しない
  → DBuffer ブレンドモードのデカールは GBuffer モード（BeforeLighting）にフォールバック
  → 静的ライティングには影響しない

r.DBuffer=1 の場合（デフォルト）:
  → DBuffer テクスチャを毎フレーム生成
  → ApplyDBufferData() がシェーダーに有効（MATERIALDECAL_DBUFFER define）
```

---

## 関連リファレンス

- [[ref_dbuffer]] — `FDBufferTextures` / `FDBufferData` 詳細
- [[a_deferred_decal]] — GBuffer デカール（BeforeLighting Stage）
