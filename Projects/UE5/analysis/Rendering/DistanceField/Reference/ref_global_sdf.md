# Ref: Global SDF — FGlobalDistanceFieldInfo / GlobalDistanceField namespace リファレンス

- 対象: `GlobalDistanceField.h`, `GlobalDistanceFieldParameters.h`, `GlobalDistanceFieldConstants.h`
- 上位: [[10_distance_field_overview]] | [[b_global_sdf]]

---

## GlobalDistanceField namespace 関数

`Engine/Source/Runtime/Renderer/Private/GlobalDistanceField.h:29`

| 関数 | シグネチャ | 説明 |
|------|-----------|------|
| `GetClipmapResolution` | `int32(bool bLumenEnabled)` | クリップマップのボクセル解像度 |
| `GetMipFactor` | `int32()` | Mip 間のスケール比 |
| `GetClipmapMipResolution` | `int32(bool bLumenEnabled)` | Mip テクスチャの解像度 |
| `GetClipmapExtent` | `float(int32 Index, const FScene*, bool)` | クリップマップ段ごとのカバー半径 |
| `GetNumGlobalDistanceFieldClipmaps` | `int32(bool, float LumenSceneViewDistance)` | クリップマップ段数（4 or 8+）|
| `GetPageAtlasSizeInPages` | `FIntVector(bool, float)` | ページアトラスのページ数 |
| `GetPageAtlasSize` | `FIntVector(bool, float)` | ページアトラスのテクセル数 |
| `GetCoverageAtlasSize` | `FIntVector(bool, float)` | カバレッジアトラスのテクセル数 |
| `GetPageTableClipmapResolution` | `uint32(bool)` | ページテーブルのクリップマップ解像度 |
| `GetPageTableTextureResolution` | `FIntVector(bool, float)` | ページテーブルテクスチャ解像度 |
| `GetMaxPageNum` | `int32(bool, float)` | 最大ページ数 |

---

## FGlobalDFCacheType

`GlobalDistanceField.h:18`

```cpp
enum FGlobalDFCacheType
{
    GDF_MostlyStatic, // 静的オブジェクト専用クリップマップ
    GDF_Full,         // 静的+動的を含む完全クリップマップ
    GDF_Num           // 2
};
```

---

## FGlobalDistanceFieldInfo

`SceneRendering.h:627`

```cpp
class FGlobalDistanceFieldInfo
{
    bool bInitialized;

    TArray<FGlobalDistanceFieldClipmap> MostlyStaticClipmaps; // [0..N-1] GDF_MostlyStatic
    TArray<FGlobalDistanceFieldClipmap> Clipmaps;             // [0..N-1] GDF_Full

    FGlobalDistanceFieldParameterData ParameterData; // → シェーダーへのバインドデータ

    // GPU テクスチャリソース
    TRefCountPtr<FRDGPooledBuffer>    PageFreeListAllocatorBuffer;
    TRefCountPtr<FRDGPooledBuffer>    PageFreeListBuffer;
    TRefCountPtr<IPooledRenderTarget> PageAtlasTexture;         // 3D: SDF 値アトラス
    TRefCountPtr<IPooledRenderTarget> CoverageAtlasTexture;     // 3D: カバレッジ
    TRefCountPtr<FRDGPooledBuffer>    PageObjectGridBuffer;
    TRefCountPtr<IPooledRenderTarget> PageTableCombinedTexture; // ページテーブル（統合）
    TRefCountPtr<IPooledRenderTarget> PageTableLayerTextures[GDF_Num]; // 種別別
    TRefCountPtr<IPooledRenderTarget> MipTexture;               // 3D: 粗い解像度 Mip
};
```

---

## FGlobalDistanceFieldParameterData

`GlobalDistanceFieldParameters.h:17`

シェーダーへバインドするデータをまとめた構造体。

```cpp
class FGlobalDistanceFieldParameterData
{
    // クリップマップ変換行列（MaxClipmaps 段分）
    FVector4f TranslatedCenterAndExtent[MaxClipmaps];      // クリップマップ中心・範囲
    FVector4f TranslatedWorldToUVAddAndMul[MaxClipmaps];   // ワールド→UV 変換（加算・乗算）
    FVector4f MipTranslatedWorldToUVScale[MaxClipmaps];    // Mip UV スケール
    FVector4f MipTranslatedWorldToUVBias[MaxClipmaps];     // Mip UV バイアス
    float     MipFactor;      // Mip 間のスケール係数
    float     MipTransition;  // Mip ブレンド遷移範囲

    // テクスチャリソース（RHI ハンドル）
    FRHITexture* PageAtlasTexture;     // Texture3D: SDF 値
    FRHITexture* CoverageAtlasTexture; // Texture3D: カバレッジ
    FRHITexture* PageTableTexture;     // Texture3D<uint>: ページテーブル
    FRHITexture* MipTexture;           // Texture3D: 粗い Mip SDF

    // スカラーパラメータ
    int32   ClipmapSizeInPages;     // 1 クリップマップのページ数
    FVector InvPageAtlasSize;       // 1 / PageAtlasSize（UV 正規化用）
    FVector InvCoverageAtlasSize;
    int32   MaxPageNum;
    float   GlobalDFResolution;     // ボクセル解像度（dimension）
    float   MaxDFAOConeDistance;    // DFAO コーントレース最大距離
    int32   NumGlobalSDFClipmaps;   // 実際のクリップマップ段数
};
```

---

## FGlobalDistanceFieldParameters2（シェーダーパラメータ構造体）

`GlobalDistanceFieldParameters.h:46`

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FGlobalDistanceFieldParameters2, )
    SHADER_PARAMETER_TEXTURE(Texture3D, GlobalDistanceFieldPageAtlasTexture)
    SHADER_PARAMETER_TEXTURE(Texture3D, GlobalDistanceFieldCoverageAtlasTexture)
    SHADER_PARAMETER_TEXTURE(Texture3D<uint>, GlobalDistanceFieldPageTableTexture)
    SHADER_PARAMETER_TEXTURE(Texture3D, GlobalDistanceFieldMipTexture)
    SHADER_PARAMETER_ARRAY(FVector4f, GlobalVolumeTranslatedCenterAndExtent, [MaxClipmaps])
    SHADER_PARAMETER_ARRAY(FVector4f, GlobalVolumeTranslatedWorldToUVAddAndMul, [MaxClipmaps])
    SHADER_PARAMETER_ARRAY(FVector4f, GlobalDistanceFieldMipTranslatedWorldToUVScale, [MaxClipmaps])
    SHADER_PARAMETER_ARRAY(FVector4f, GlobalDistanceFieldMipTranslatedWorldToUVBias, [MaxClipmaps])
    SHADER_PARAMETER(float, GlobalDistanceFieldMipFactor)
    SHADER_PARAMETER(float, GlobalDistanceFieldMipTransition)
    SHADER_PARAMETER(int32, GlobalDistanceFieldClipmapSizeInPages)
    SHADER_PARAMETER(FVector3f, GlobalDistanceFieldInvPageAtlasSize)
    SHADER_PARAMETER(FVector3f, GlobalDistanceFieldInvCoverageAtlasSize)
    SHADER_PARAMETER(float, GlobalVolumeDimension)
    SHADER_PARAMETER(float, GlobalVolumeTexelSize)
    SHADER_PARAMETER(float, MaxGlobalDFAOConeDistance)
    SHADER_PARAMETER(uint32, NumGlobalSDFClipmaps)
    // ... カバレッジ係数, サンプラー等
END_SHADER_PARAMETER_STRUCT()
```

変換ユーティリティ: `SetupGlobalDistanceFieldParameters(ParameterData)` → `FGlobalDistanceFieldParameters2`

---

## UpdateGlobalDistanceFieldVolume

`GlobalDistanceField.h:50`

```cpp
extern void UpdateGlobalDistanceFieldVolume(
    FRDGBuilder& GraphBuilder,
    FRDGExternalAccessQueue& ExternalAccessQueue,
    FViewInfo& View,
    FScene* Scene,
    float MaxOcclusionDistance,  // DFAO 最大距離（クリップマップ範囲に影響）
    bool bLumenEnabled,          // Lumen 有効ならクリップマップ段数増
    FGlobalDistanceFieldInfo& Info); // 出力：更新済み Info（ParameterData 含む）
```

---

## 補助関数

| 関数 | ファイル | 説明 |
|------|---------|------|
| `UseGlobalDistanceField()` | `GlobalDistanceField.h:26` | Global SDF が有効かを判定 |
| `UseGlobalDistanceField(Parameters)` | `GlobalDistanceField.h:27` | パラメータを参照した判定 |
| `SetupGlobalDistanceFieldParameters(Data)` | `GlobalDistanceFieldParameters.h:74` | CPU→GPU バインド変換 |

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.AOGlobalDistanceField` | Global SDF 有効フラグ |
| `r.GlobalDistanceField.MaxPageNum` | ページアトラスの最大ページ数 |
| `r.GlobalDistanceField.ClipmapExtent` | クリップマップ範囲係数 |
| `r.GlobalDistanceField.Resolution` | ボクセル解像度オーバーライド |
