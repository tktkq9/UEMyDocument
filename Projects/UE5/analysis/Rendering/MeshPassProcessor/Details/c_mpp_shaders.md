# C: FMeshMaterialShader とシェーダーバインディング

- 対象: `MeshMaterialShader.h`, `MeshDrawShaderBindings.h`
- 上位: [[11_mpp_overview]]
- Reference: [[ref_mpp_shaders]]

---

## FMeshMaterialShader — メッシュシェーダー基底

マテリアルと頂点ファクトリーの組み合わせで特殊化されるシェーダーの基底クラス。  
HLSL 側の対応は `MeshMaterialShader.ush`。

```cpp
class FMeshMaterialShader : public FMaterialShader
{
public:
    // 頂点ファクトリーからシェーダーを取得する型付きラッパー
    template<typename ShaderType>
    static TShaderRef<ShaderType> GetShaderFromMaterial(
        const FMaterial& Material,
        const FVertexFactory* VertexFactory);

    // シェーダー定義マクロ（サブクラスで使用）
    // DECLARE_SHADER_TYPE(FMyMeshVS, MeshMaterial)
    // IMPLEMENT_MATERIAL_SHADER_TYPE(,FMyMeshVS, TEXT("/Engine/Private/MyShader.usf"), TEXT("MainVS"), SF_Vertex)

    // 頂点ファクトリーのパラメータをバインドする
    void GetShaderBindings(
        const FScene* Scene,
        ERHIFeatureLevel::Type FeatureLevel,
        const FPrimitiveSceneProxy* PrimitiveSceneProxy,
        const FMaterialRenderProxy& MaterialRenderProxy,
        const FMaterial& Material,
        const FMeshPassProcessorRenderState& DrawRenderState,
        const FMeshMaterialShaderElementData& ShaderElementData,
        FMeshDrawSingleShaderBindings& ShaderBindings) const;
};
```

### シェーダー定義の例

```cpp
// ヘッダ
class FMyMeshVS : public FMeshMaterialShader
{
    DECLARE_SHADER_TYPE(FMyMeshVS, MeshMaterial);
public:
    static bool ShouldCompilePermutation(
        const FMeshMaterialShaderPermutationParameters& Parameters)
    {
        return IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM5);
    }
    static void ModifyCompilationEnvironment(
        const FMaterialShaderPermutationParameters& Parameters,
        FShaderCompilerEnvironment& OutEnvironment)
    {
        FMeshMaterialShader::ModifyCompilationEnvironment(Parameters, OutEnvironment);
        OutEnvironment.SetDefine(TEXT("MY_FEATURE"), 1);
    }
};

// CPP
IMPLEMENT_MATERIAL_SHADER_TYPE(, FMyMeshVS,
    TEXT("/Engine/Private/MyShader.usf"), TEXT("MainVS"), SF_Vertex);
```

---

## FMeshProcessorShaders — シェーダーセット

1 パスで使用するシェーダーをまとめる構造体テンプレート:

```cpp
// 典型的な Raster パス用のシェーダーセット定義
template<
    typename VertexShaderType,
    typename PixelShaderType,
    typename GeometryShaderType = FMeshMaterialShader, // デフォルト=なし
    typename HullShaderType     = FMeshMaterialShader,
    typename DomainShaderType   = FMeshMaterialShader>
struct TMeshProcessorShaders
{
    TShaderRef<VertexShaderType>   VertexShader;
    TShaderRef<GeometryShaderType> GeometryShader;
    TShaderRef<HullShaderType>     HullShader;
    TShaderRef<DomainShaderType>   DomainShader;
    TShaderRef<PixelShaderType>    PixelShader;
};

// 使用例
TMeshProcessorShaders<FBasePassVS, FBasePassPS> PassShaders;
PassShaders.VertexShader = Material.GetShader<FBasePassVS>(VFType, /* PermutationId */ 0);
PassShaders.PixelShader  = Material.GetShader<FBasePassPS>(/* PermutationId */ 0);
```

---

## FMeshDrawShaderBindings — バインディングストレージ

シェーダーごとのリソースバインディングを格納する密なバイト配列。  
`BuildMeshDrawCommands()` 内部で `FMeshDrawSingleShaderBindings` 経由で書き込む。

```cpp
class FMeshDrawShaderBindings
{
public:
    // シェーダーごとのバインディング領域を確保
    void Initialize(const FMeshProcessorShaders& Shaders);

    // 各シェーダーのバインディングへのアクセサ
    FMeshDrawSingleShaderBindings GetSingleShaderBindings(
        EShaderFrequency Frequency, int32& DataOffset);

    // RHI コマンドリストへの適用
    void SetOnCommandList(FRHICommandList& RHICmdList, ...) const;

    // 異なるバインディングかどうかの比較（動的インスタンシング判定）
    bool operator==(const FMeshDrawShaderBindings& Rhs) const;
};
```

---

## FMeshDrawSingleShaderBindings — 単一シェーダーのバインディング

```cpp
class FMeshDrawSingleShaderBindings : public FMeshDrawShaderBindingsLayout
{
public:
    // Uniform Buffer バインディング
    template<typename UniformBufferStructType>
    void Add(const TShaderUniformBufferParameter<UniformBufferStructType>& Param,
             const TUniformBufferRef<UniformBufferStructType>& UB);

    // SRV バインディング
    void Add(FShaderResourceParameter Param, FRHIShaderResourceView* SRV);

    // サンプラーバインディング
    void Add(FShaderResourceParameter SamplerParam, FRHISamplerState* Sampler);

    // テクスチャバインディング
    void Add(FShaderResourceParameter TexParam, FRHITexture* Texture);

    // スカラー/ベクターバインディング（ルート定数）
    template<class ParameterType>
    void Add(FShaderParameter Param, const ParameterType& Value);
};
```

### バインディング設定の例

```cpp
// BuildMeshDrawCommands → GetShaderBindings の中で呼ばれる典型的なコード
void FMyMeshVS::GetShaderBindings(
    ..., FMeshDrawSingleShaderBindings& ShaderBindings) const
{
    // Uniform Buffer
    ShaderBindings.Add(GetUniformBufferParameter<FViewUniformShaderParameters>(),
        View->ViewUniformBuffer);

    // SRV
    ShaderBindings.Add(MyTexture, MaterialResource.GetTextureSRV());

    // サンプラー
    ShaderBindings.Add(MyTextureSampler, TStaticSamplerState<SF_Linear>::GetRHI());

    // スカラー値
    ShaderBindings.Add(MyFloat, 1.0f);
}
```

---

## FMeshMaterialShaderElementData — シェーダー要素データ

`BuildMeshDrawCommands()` の `ShaderElementData` 引数として渡す追加データ:

```cpp
struct FMeshMaterialShaderElementData
{
    uint32 NumTexCoords;                     // テクスチャ座標チャンネル数
    float  LODFadeAlpha;                     // LOD フェードアルファ
    float  LODFadeOneMinusAlpha;
    float  ReverseCulling;                   // カリング反転フラグ
    bool   bDitheredLODTransition;
    bool   bHoveredWithWidgetAxis;           // エディタウィジェット上にある場合
};
```

---

## VertexFactory との連携

MeshMaterialShader は `FVertexFactory` と組み合わせてコンパイルされる。

```cpp
// VertexFactory タイプの取得
const FVertexFactoryType* VFType = MeshBatch.VertexFactory->GetType();

// VertexFactory 固有シェーダー取得
TShaderRef<FMyVS> VS = Material.GetShader<FMyVS>(VFType, PermutationId);

// 主な VertexFactory タイプ
// - FLocalVertexFactory       : 通常スタティックメッシュ
// - FGPUSkinVertexFactory     : スケルタルメッシュ
// - FNaniteVertexFactory      : Nanite メッシュ
// - FLandscapeVertexFactory   : ランドスケープ
// - FSplineMeshVertexFactory  : スプラインメッシュ
```

---

## コード実行フロー

### エントリポイント

```
BuildMeshDrawCommands<PassShadersType, ShaderElementDataType>()
  │
  ├─ FMeshDrawCommand::InitializeShaderBindings(PassShaders)
  │    各シェーダーのバインディング領域（バイト配列）を確保
  │
  ├─ [VertexShader バインディング]
  │    GetSingleShaderBindings(SF_Vertex, DataOffset)
  │      → FMeshDrawSingleShaderBindings vsBindings
  │    PassShaders.VertexShader->GetShaderBindings(
  │        Scene, FeatureLevel, Proxy, MatProxy, Mat,
  │        DrawRenderState, ShaderElementData, vsBindings)
  │      │
  │      ├─ FMeshMaterialShader::GetShaderBindings()
  │      │    View UB、Primitive UB を Add()
  │      │
  │      └─ FVertexFactory::GetShaderBindings()
  │           VertexFactory 固有パラメータを Add()
  │
  ├─ [PixelShader バインディング]
  │    GetSingleShaderBindings(SF_Pixel, DataOffset)
  │    PassShaders.PixelShader->GetShaderBindings(...)
  │      └─ マテリアルテクスチャ・サンプラーを Add()
  │
  └─ (GeometryShader / HullShader / DomainShader も同様)
```

### フロー詳細

1. **InitializeShaderBindings()** `MeshPassProcessor.h`  
   `FMeshProcessorShaders` を受け取り、各シェーダーステージ（SF_Vertex, SF_Pixel 等）に必要なバイト数を計算してインライン配列または Heap 配列を確保する。

2. **GetSingleShaderBindings()** `MeshPassProcessor.h:978`  
   ステージごとに `FMeshDrawSingleShaderBindings` ビューを返す。`DataOffset` を内部で進めることで、各ステージが独自の連続メモリ領域に書き込む。

3. **FMeshMaterialShader::GetShaderBindings()** `MeshMaterialShader.h`  
   - `View` の `ViewUniformBuffer` を Add()
   - `PrimitiveSceneProxy` の `PrimitiveUniformBuffer` を Add()
   - `DrawRenderState` の PassUniformBuffer を Add()
   - `ShaderElementData` から LOD フェード値等を Add()

4. **FVertexFactory::GetShaderBindings()** （各 VF 実装）  
   VertexFactory 固有のバッファ（ボーン行列 UB、スプラインパラメータ等）を Add()。呼び出しは `VertexFactoryParameters.GetElementShaderBindings()` 経由。

5. **派生シェーダーの追加バインディング**  
   `FBasePassVS::GetShaderBindings()` 等、具体的なシェーダークラスが `Super::GetShaderBindings()` を呼んだ後に追加リソースを Add() する。

6. **SetOnCommandList()** `MeshPassProcessor.h:1008`  
   `SubmitDrawBegin()` から呼ばれ、蓄積されたバインディングを `FRHICommandList` に一括適用する。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 説明 |
|--------------|---------|------|
| `FMeshMaterialShader::GetShaderBindings()` | `MeshMaterialShader.h` | View/Primitive UB のバインド |
| `FMeshDrawShaderBindings::Initialize()` | `MeshPassProcessor.h` | バインディング領域確保 |
| `FMeshDrawShaderBindings::GetSingleShaderBindings()` | `MeshPassProcessor.h:978` | ステージ別 Bindings ビュー取得 |
| `FMeshDrawSingleShaderBindings::Add()` | `MeshPassProcessor.h` | リソースをバインディング領域に書き込み |
| `FMeshDrawShaderBindings::SetOnCommandList()` | `MeshPassProcessor.h:1008` | バインディングを RHI コマンドに適用 |
| `FMeshDrawShaderBindings::MatchesForDynamicInstancing()` | `MeshPassProcessor.cpp:1018` | 動的インスタンシング可否の比較 |
| `FVertexFactory::GetShaderBindings()` | 各 VF 実装 | VertexFactory 固有リソースのバインド |
| `DECLARE_SHADER_TYPE` / `IMPLEMENT_MATERIAL_SHADER_TYPE` | `MeshMaterialShader.h` | シェーダークラスのメタデータ登録マクロ |
