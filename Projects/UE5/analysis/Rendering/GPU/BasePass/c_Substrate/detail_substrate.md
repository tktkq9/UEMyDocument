# Substrate GBuffer エンコード シェーダー詳細

- グループ: c - Substrate
- GPU 概要: [[01_basepass_gpu_overview]]
- CPU 詳細: [[c_substrate_gbuffer]]
- リファレンス: [[ref_substrate]]

---

## 概要

**Substrate** は UE5 の次世代マテリアルシステム。従来の固定 ShadingModel ID ではなく、  
**BSDF スラブ（SLAB）** の組み合わせで複雑なマテリアルを表現する。  
GBuffer 書き込み時は `SubstrateCompressedBSDF` 等のエンコード関数でパッキングされ、  
Deferred Lighting パスで `SubstrateFetchMaterial()` でデコードされる。

---

## レンダリングパスの構成

```
BasePass（Substrate 有効）
  │
  ├─ VS: BasePassVertexShader.usf#Main()（通常 BasePass と同じ）
  └─ PS: BasePassPixelShader.usf
      SUBSTRATE_ENABLED=1 時、GBuffer 書き込み前に以下が実行される:
      │
      ├─ SubstratePixelHeader 初期化
      ├─ BSDF ツリーの展開（マテリアルグラフの各 BSDF ノードを評価）
      ├─ SubstrateCompressedBSDFCompress() で GBuffer エンコード
      │   → SubstrateBuffer（独自フォーマット）または従来 GBuffer へ変換
      └─ SubstrateConvertToLegacyGBuffer()（非 Substrate GBuffer フォーマット時）
```

---

## Substrate GBuffer フォーマット

### SUBSTRATE_GBUFFER_FORMAT = 1（Substrate 専用）

| ターゲット | 内容 |
|-----------|------|
| SubstrateBuffer0 | BSDF0: BaseColor.rgb + Weight |
| SubstrateBuffer1 | BSDF0: Normal（Octahedron）+ Roughness |
| SubstrateBuffer2 | BSDF1 以降 または SpecularColor / Anisotropy |
| SubstrateHeader | ShadingModelFlags / BSDF 数 / Layer 情報 |

### SUBSTRATE_GBUFFER_FORMAT = 0（従来 GBuffer 互換）

通常の GBufferA/B/C/D に Substrate データを変換して書き込む（`SubstrateConvertToLegacyGBuffer()`）。

---

## シェーダーコアロジック（Substrate.ush より）

### BSDF ツリー評価

```hlsl
// BasePassPixelShader.usf 内（Substrate モード）
FSubstrateData SubstrateData = GetSubstrateData(MaterialParameters, PixelMaterialInputs, ...);
// → SubstrateData.Slabs[] に各 BSDF レイヤーが格納される

// Slab の例（DefaultLit 相当）:
FSubstrateSlab Slab;
Slab.DiffuseAlbedo = BaseColor * (1 - Metallic);
Slab.F0 = lerp(0.04, BaseColor, Metallic);  // フレネル0度反射率
Slab.Roughness = Roughness;
Slab.Normal = Normal;
```

### SubstrateCompressedBSDFCompress（GBuffer エンコード）

```hlsl
// Substrate.ush
void SubstrateCompressedBSDFCompress(
    FSubstrateData SubstrateData,
    out FSubstrateGBufferData GBufferData)
{
    // 複数 BSDF を圧縮してエンコード
    // Layer 0: メインの BSDF（最も重要なレイヤー）
    // Layer 1〜N: 追加レイヤー（Clearcoat 等）
    EncodeSubstrateSlab(SubstrateData.Slabs[0], GBufferData.BSDF0);
    // ...
}
```

---

## CPU 呼び出しの流れ

```
RenderBasePass()                                 // SceneRendering.cpp
  │
  ├─ SUBSTRATE_ENABLED=1 時:
  │   SubstrateSceneData.SubstrateBufferOffset を View.SubstrateViewData にバインド
  │   → BasePassPixelShader が SubstrateBuffer UAV に書き込み
  │
  └─ Deferred Lighting パスでデコード:
      SubstrateFetchMaterial(SubstrateBuffer, ScreenUV)
      → FSubstrateData を復元してライティング計算に使用
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `Substrate/Substrate.ush` | ヘッダ | BSDF 構造体・エンコード・デコード全般 |
| `Substrate/SubstrateExport.ush` | ヘッダ | GBuffer 書き込みの最終出力処理 |
| `Substrate/SubstrateDBuffer.ush` | ヘッダ | DBuffer デカールの Substrate 対応 |
| `Substrate/SubstrateTile.ush` | ヘッダ | タイル分類（複雑な Substrate のみ後処理）|
