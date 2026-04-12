# Lumen Card Capture シェーダー詳細

- グループ: a - Surface Cache
- GPU 概要: [[01_gpu_overview]]
- CPU 詳細: [[a_lumen_surface_cache]]
- リファレンス: [[ref_card_shaders]]

---

## 概要

Lumen の「Surface Cache」を構築する最初のステップ。  
シーン内の各オブジェクトを **Card（平面的な簡略表現）** として専用のテクスチャアトラスに描画する。  
通常の GBuffer 描画と似ているが、**WPO（World Position Offset）を無効化**し、  
マテリアルの Albedo / Normal / Emissive だけを書き出す専用パスとして動作する。

---

## レンダリングパスの構成

```
Card Capture パス
  │
  ├─ [通常メッシュ] Rasterize Pass
  │   VS: LumenCardVertexShader.usf#Main()
  │   PS: LumenCardPixelShader.usf#Main()
  │   RT: [0] Albedo+Opacity  [1] Normal+Roughness  [2] Emissive
  │
  └─ [Nanite メッシュ] Compute Shading Pass
      CS: LumenCardComputeShader.usf#Main()
      VisBuffer を読み取り → マテリアル評価 → Atlas に書き込み
```

---

## 入出力

### 入力

| リソース | 説明 |
|---------|------|
| `VertexFactory` | メッシュのジオメトリ入力（LocalVertexFactory / SkinVertexFactory 等） |
| `LumenCardPass.SceneTextures` | SceneTextures ユニフォームバッファ（シェーダーテクスチャアクセス用）|
| `NaniteShading.VisBuffer64` | Nanite の VisibilityBuffer（CS 版のみ）|
| `Material.ush`（Generated）| マテリアルグラフから生成されたシェーダーコード |

### 出力（Surface Cache アトラス）

| レンダーターゲット | フォーマット | 内容 |
|----------------|------------|------|
| `Target0`（Albedo Atlas） | `RGBA8` | `float4(Albedo.rgb, Opacity)` |
| `Target1`（Normal Atlas） | `RGBA8` | `float4(Normal.xyz * 0.5 + 0.5, Roughness)` |
| `Target2`（Emissive Atlas）| `R11G11B10F` | `float4(Emissive.rgb, 0)` |

---

## シェーダーコアロジック（LumenCardBasePass.ush）

3枚の RT への書き込みは `LumenCardBasePass.ush` の `LumenCardBasePass()` で統一されている。

```hlsl
FLumenCardBasePassOutput LumenCardBasePass(
    FPixelMaterialInputs PixelMaterialInputs,
    FMaterialPixelParameters MaterialParameters,
    float4 SvPosition)
{
    // Substrate 有効時: BSDF ツリーを展開して Diffuse + Specular を平均化
    // 従来マテリアル: GetMaterialDiffuseColor() / GetMaterialNormal() 等を呼ぶ

    FLumenCardBasePassOutput Output;
    Output.Target0 = float4(DiffuseColor, Opacity);       // Albedo
    Output.Target1 = float4(EncodeNormal(Normal), Roughness); // Normal
    Output.Target2 = float4(Emissive, 0);                 // Emissive
    return Output;
}
```

**注意点：**
- Specular は保存しない（Surface Cache は Diffuse のみ）
- WPO（`GetMaterialWorldPositionOffset()`）は意図的に**コメントアウト**（`LumenCardVertexShader.usf:41`）
- Two-sided マテリアルは `bIsFrontFace = false` で固定（Nanite CS 版）

---

## Nanite CS 版の特徴（LumenCardComputeShader.usf）

通常 VS/PS の代わりに Nanite の VisibilityBuffer からピクセルを復元して実行する。

```
[numthreads(64, 1, 1)]
Main(ThreadIndex, GroupID)
  │
  ├─ PackedTile から CardIndex・PixelPos を復元（Morton デコード）
  ├─ VisBuffer64[PixelPos] からクラスター・トライアングルを取得
  ├─ ShadingBin が一致するピクセルのみ ProcessPixel() を呼ぶ
  └─ ProcessPixel() → LumenCardBasePass() → ExportPixel() で Atlas に書き込み
```

---

## Surface Cache の後処理（LumenSurfaceCache.usf）

Card Capture 後、アトラスデータを圧縮・コピーする後処理 CS 群が続く。

| エントリポイント | 役割 |
|---------------|------|
| `CopyCapturedCardPageCS()` | キャプチャ結果を物理アトラスページにコピー |
| `ClearCompressedAtlasCS()` | 使われなくなったページをクリア |
| `GenerateDilationTileDataCS()` | ページ境界のダイレーション（エッジのにじみ防止）|
| `LumenCardCopyPS()` | アトラスを直接コピーする PS 版（Emissive 等の特殊処理）|
| `LumenCardResamplePS()` | アトラスのリサンプリング（解像度変更時）|

---

## CPU 呼び出しの流れ

```
UpdateLumenScene()                     // LumenSceneRendering.cpp:2490
  │
  └─ LumenScene::RenderCardCaptureViews()
       │
       ├─ [通常メッシュ]
       │   RenderMeshCommands(GraphBuilder, …, LumenCardPass)
       │     → DrawIndexedPrimitive
       │       VS: LumenCardVertexShader.usf
       │       PS: LumenCardPixelShader.usf
       │
       └─ [Nanite メッシュ]
           Nanite::BuildShadingCommands(… ENaniteMeshPass::LumenCardCapture …)
           Nanite::DispatchShading(… LumenCardComputeShader …)
             CS: LumenCardComputeShader.usf
```

**CPU 側でのパラメーター割り当て：**

```cpp
// LumenSceneRendering.cpp 内
FLumenCardPassUniformParameters CardPassUniformParameters;
CardPassUniformParameters.SceneTextures = SceneTextures.UniformBuffer;
// → シェーダー側: LumenCardPass.SceneTextures として参照
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `LumenCardVertexShader.usf` | VS | Card ジオメトリ変換（WPO なし）|
| `LumenCardPixelShader.usf` | PS | マテリアル評価 → 3RT 書き込み |
| `LumenCardComputeShader.usf` | CS | Nanite VisBuffer から同等処理 |
| `SurfaceCache/LumenSurfaceCache.usf` | CS/PS | アトラスのコピー・圧縮・ダイレーション |
| `LumenCardBasePass.ush` | ヘッダ | PS/CS 共通のシェーディングロジック |
| `LumenCardCommon.ush` | ヘッダ | Card データ構造・インデックス計算 |
| `LumenMaterial.ush` | ヘッダ | Surface Cache 用マテリアル評価ユーティリティ |
