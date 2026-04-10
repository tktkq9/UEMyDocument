# REF: VirtualShadowMapShaders.h

- 対象ファイル: `Private/VirtualShadowMaps/VirtualShadowMapShaders.h`
- 関連Details: [[d_vsm_projection]]

---

## 役割

VSM 用グローバルシェーダーの**基底クラス定義ファイル**。  
すべての VSM コンピュートシェーダー・グラフィクスシェーダーはこれを継承する。  
コンパイル条件の統一と HLSL 2021 の有効化が主な目的。

---

## クラス

### `FVirtualShadowMapGlobalShader`

**`FGlobalShader` 継承**。VSM 用シェーダーの最上位基底クラス。

```cpp
class FVirtualShadowMapGlobalShader : public FGlobalShader
{
public:
    // Renderer モジュール内でのみコンパイル（プラットフォーム要件確認）
    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters);

    // HLSL 2021 を有効化（C++20 ライクなテンプレート・演算子オーバーロード対応）
    static void ModifyCompilationEnvironment(
        const FGlobalShaderPermutationParameters& Parameters,
        FShaderCompilerEnvironment& OutEnvironment);
};
```

**`ShouldCompilePermutation` の条件:**
- `IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM5)`（SM5以上）
- `RHISupportsRayTracing` 等のプラットフォーム要件

---

### `FVirtualShadowMapPageManagementShader`

**`FVirtualShadowMapGlobalShader` 継承**。ページ管理コンピュートシェーダーの基底クラス。

```cpp
class FVirtualShadowMapPageManagementShader : public FVirtualShadowMapGlobalShader
{
public:
    // 2D ディスパッチ用グループサイズ（BeginMarkPages 等で使用）
    static constexpr int32 DefaultCSGroupXY = 8;   // 8×8 = 64スレッド/グループ

    // 1D ディスパッチ用グループサイズ（BuildPageAllocations 等で使用）
    static constexpr int32 DefaultCSGroupX  = 256;

    // グループサイズ定数をシェーダーマクロとして注入
    static void ModifyCompilationEnvironment(
        const FGlobalShaderPermutationParameters& Parameters,
        FShaderCompilerEnvironment& OutEnvironment);
    // → VIRTUAL_SHADOW_MAP_PAGE_MANAGEMENT_SHADER_GROUP_SIZE_XY = 8
    // → VIRTUAL_SHADOW_MAP_PAGE_MANAGEMENT_SHADER_GROUP_SIZE_X = 256
};
```

---

## 使用例（継承パターン）

```cpp
// ページマーキングコンピュートシェーダーの例（.cpp 側）
class FMarkPagesFrustumCullCS : public FVirtualShadowMapPageManagementShader
{
    DECLARE_GLOBAL_SHADER(FMarkPagesFrustumCullCS);
    SHADER_USE_PARAMETER_STRUCT(FMarkPagesFrustumCullCS, FVirtualShadowMapPageManagementShader);

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_RDG_TEXTURE(Texture2D, SceneDepth)
        SHADER_PARAMETER_STRUCT_REF(FVirtualShadowMapUniformParameters, VirtualShadowMap)
        // ... 各種パラメータ
        SHADER_PARAMETER_RDG_BUFFER_UAV(RWBuffer<uint>, PageRequestFlags)
    END_SHADER_PARAMETER_STRUCT()
};

// ディスパッチ:
// FComputeShaderUtils::AddPass(GraphBuilder, ...,
//     TShaderMapRef<FMarkPagesFrustumCullCS>(View.ShaderMap),
//     PassParameters,
//     FIntVector(
//         FMath::DivideAndRoundUp(ViewRect.Width(),  FMarkPagesFrustumCullCS::DefaultCSGroupXY),
//         FMath::DivideAndRoundUp(ViewRect.Height(), FMarkPagesFrustumCullCS::DefaultCSGroupXY),
//         1))
```

---

## 備考

このファイルには CVar は定義されていない。  
定数 `DefaultCSGroupXY = 8` / `DefaultCSGroupX = 256` はコンパイル時定数であり、  
実行時変更不可。シェーダー側では以下のマクロとして参照される:

```hlsl
// HLSL 側でのグループサイズ参照
[numthreads(VIRTUAL_SHADOW_MAP_PAGE_MANAGEMENT_SHADER_GROUP_SIZE_XY,
            VIRTUAL_SHADOW_MAP_PAGE_MANAGEMENT_SHADER_GROUP_SIZE_XY, 1)]
void MarkPagesCS(...) { ... }
```
