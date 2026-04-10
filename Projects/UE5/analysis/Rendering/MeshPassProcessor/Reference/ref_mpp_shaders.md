# REF: MeshPassProcessor シェーダー関連型

- 対象ファイル: `MeshMaterialShader.h`, `MeshDrawShaderBindings.h`, `MeshMaterialShader.h`
- 関連Details: [[c_mpp_shaders]]

---

## FMeshMaterialShader — 継承ツリー

```
FShader
└─ FMaterialShader
   └─ FMeshMaterialShader
      ├─ FBasePassVS / FBasePassPS
      ├─ FDepthOnlyVS / FDepthOnlyPS
      ├─ FShadowDepthVS / FShadowDepthPS
      └─ （各パス固有シェーダー）
```

---

## FMeshMaterialShader — クラス定義

```cpp
class FMeshMaterialShader : public FMaterialShader
{
public:
    // シェーダーコンパイル条件（サブクラスでオーバーライド）
    static bool ShouldCompilePermutation(
        const FMeshMaterialShaderPermutationParameters& Parameters);

    // コンパイル環境カスタマイズ
    static void ModifyCompilationEnvironment(
        const FMaterialShaderPermutationParameters& Parameters,
        FShaderCompilerEnvironment& OutEnvironment);

    // バインディング設定（サブクラスでオーバーライドして追加バインドを設定）
    void GetShaderBindings(
        const FScene* Scene,
        ERHIFeatureLevel::Type FeatureLevel,
        const FPrimitiveSceneProxy* PrimitiveSceneProxy,
        const FMaterialRenderProxy& MaterialRenderProxy,
        const FMaterial& Material,
        const FMeshPassProcessorRenderState& DrawRenderState,
        const FMeshMaterialShaderElementData& ShaderElementData,
        FMeshDrawSingleShaderBindings& ShaderBindings) const;

    // シェーダーのパラメータ定義
    LAYOUT_FIELD(FVertexFactoryShaderParameters, VertexFactoryParameters);
};

// 定義マクロ
#define DECLARE_SHADER_TYPE(ShaderClass, ShaderMetaTypeShortcut)
#define IMPLEMENT_MATERIAL_SHADER_TYPE(TemplatePrefix, ShaderClass, SourceFilename, FunctionName, Frequency)
```

---

## FMeshMaterialShaderPermutationParameters

```cpp
struct FMeshMaterialShaderPermutationParameters : FMaterialShaderPermutationParameters
{
    const FVertexFactoryType* const VertexFactoryType;

    FMeshMaterialShaderPermutationParameters(
        EShaderPlatform InPlatform,
        const FMaterialShaderParameters& InMaterialParameters,
        const FVertexFactoryType* InVertexFactoryType,
        int32 InPermutationId);
};
```

---

## FMeshMaterialShaderElementData

```cpp
struct FMeshMaterialShaderElementData
{
    uint32 NumTexCoords;
    float  LODFadeAlpha;
    float  LODFadeOneMinusAlpha;
    float  ReverseCulling;             // -1.0 または 1.0
    bool   bDitheredLODTransition;
    bool   bHoveredWithWidgetAxis;     // エディタウィジェットホバー

    FMeshMaterialShaderElementData()
        : NumTexCoords(0)
        , LODFadeAlpha(0.0f)
        , LODFadeOneMinusAlpha(1.0f)
        , ReverseCulling(1.0f)
        , bDitheredLODTransition(false)
        , bHoveredWithWidgetAxis(false)
    {}
};
```

---

## TMeshProcessorShaders — シェーダーセット

```cpp
struct FMeshProcessorShaders
{
    TShaderRef<FShader> VertexShader;
    TShaderRef<FShader> HullShader;
    TShaderRef<FShader> DomainShader;
    TShaderRef<FShader> PixelShader;
    TShaderRef<FShader> GeometryShader;
    TShaderRef<FShader> ComputeShader;  // Compute パスの場合

    bool IsValid() const { return VertexShader.IsValid() || ComputeShader.IsValid(); }
    uint32 GetDynamicInstancingHash() const;

    bool operator==(const FMeshProcessorShaders& Other) const;
};

// 型付きテンプレート版（各シェーダーに型情報を持つ）
template<
    typename VertexShaderType,
    typename PixelShaderType = FMeshMaterialShader,
    typename GeometryShaderType = FMeshMaterialShader,
    typename HullShaderType = FMeshMaterialShader,
    typename DomainShaderType = FMeshMaterialShader>
struct TMeshProcessorShaders
{
    TShaderRef<VertexShaderType>   VertexShader;
    TShaderRef<GeometryShaderType> GeometryShader;
    TShaderRef<HullShaderType>     HullShader;
    TShaderRef<DomainShaderType>   DomainShader;
    TShaderRef<PixelShaderType>    PixelShader;

    operator FMeshProcessorShaders() const { ... }
};
```

---

## FMeshDrawShaderBindingsLayout

```cpp
class FMeshDrawShaderBindingsLayout
{
public:
    const FShaderParameterMapInfo& ParameterMapInfo;

    uint32 GetDataSizeBytes()      const;
    uint32 GetLooseDataSizeBytes() const;

    bool   HasParameterBindings()  const;
};
```

---

## FVertexFactory 主要型

| クラス | 用途 |
|--------|------|
| `FLocalVertexFactory` | スタティックメッシュ（通常） |
| `FGPUSkinVertexFactory` | スケルタルメッシュ（ボーンスキニング）|
| `FNaniteVertexFactory` | Nanite ジオメトリ |
| `FLandscapeVertexFactory` | ランドスケープ |
| `FSplineMeshVertexFactory` | スプラインメッシュ |
| `FParticleVertexFactory` | パーティクル |
| `FInstancedStaticMeshVertexFactory` | インスタンス化スタティックメッシュ |

---

## EShaderFrequency（シェーダーステージ）

```cpp
enum EShaderFrequency : uint8
{
    SF_Vertex   = 0,   // 頂点シェーダー
    SF_Hull     = 1,   // ハルシェーダー（テッセレーション）
    SF_Domain   = 2,   // ドメインシェーダー
    SF_Pixel    = 3,   // ピクセルシェーダー
    SF_Geometry = 4,   // ジオメトリシェーダー
    SF_Compute  = 5,   // コンピュートシェーダー
    SF_RayGen   = 6,   // レイジェネレーション
    // ...
    SF_NumFrequencies,
    SF_NumGraphicsFrequencies = SF_Compute,
};
```
