# リファレンス：NaniteMaterialsSceneExtension.h / NaniteMaterialsSceneExtension.cpp

- グループ: b - Materials & Shading
- 上位: [[b_nanite_materials_shading]]
- 関連: [[ref_nanite_materials]] | [[ref_nanite_shading]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteMaterialsSceneExtension.h/.cpp`

---

## 概要

マテリアルスロット等のプリミティブごとのデータを GPU バッファに非同期アップロードし、  
レンダリングパスで参照できる状態に保つ `ISceneExtension` 実装。  
プリミティブの追加・削除・変形に応じて差分更新を行い、  
バッファのデフラグも管理する。

---

## 主要クラス

```cpp
namespace Nanite
{

// ISceneExtension 実装（シーンに紐付く）
class FMaterialsSceneExtension : public ISceneExtension
{
public:
    static bool ShouldCreateExtension(const FScene& Scene);

    // マテリアル GPU バッファ（永続）
    FMaterialBuffers Buffers;

    // バッファデフラグ処理
    void ProcessBufferDefragmentation(FRDGBuilder&);

    // シェーディングコマンド構築後のコールバック
    void PostBuildNaniteShadingCommands(
        FRDGBuilder&,
        const FNaniteShadingPipelines&);

    // ヒットプロキシ ID バッファの作成（エディタ用）
    FRDGBufferRef CreateHitProxyIDBuffer(FRDGBuilder&);

    // デバッグビューモードバッファの作成
    FRDGBufferRef CreateDebugViewModeBuffer(FRDGBuilder&);

    // --- ISceneExtension サブクラス ---
    class FUpdater : public ISceneExtensionUpdater
    {
        // プリミティブ追加時に FPackedPrimitiveData を生成
        void PreSceneUpdate(FScenePreUpdateChangeSet&) override;
        void PostSceneUpdate(FScenePostUpdateChangeSet&) override;
    };

    class FRenderer : public ISceneExtensionRenderer
    {
        // バッファのアップロードパスを RDG に追加
        void PostRenderOpaque(FPostRenderOpaqueInputs&) override;
    };
};

// プリミティブデータ（GPU バッファにアップロードする形式）
struct FPackedPrimitiveData
{
    uint32 PrimitiveIndex;
    TArray<FNaniteMaterialSlot> MaterialSlots;
    // HitProxy ID, DebugViewMode 等
};

// 永続マテリアルバッファ（フレームをまたいで保持）
struct FMaterialBuffers
{
    FRWBufferStructured MaterialSlotBuffer;   // FNaniteMaterialSlot 配列
    FRWBufferStructured ShadingBinBuffer;     // FNaniteShadingBin 配列
    uint32 NumAllocatedSlots;
};

// Scatter アップロード（差分書き込み）
class FUploader
{
    void Lock(uint32 NumPrimitivesToUpload);
    void Add(const FPackedPrimitiveData& Data);
    void Unlock(FRDGBuilder&);
};

} // namespace Nanite
```

---

## 主要関数

| 関数 | 説明 |
|------|------|
| `ShouldCreateExtension(FScene&)` | Nanite が有効なプラットフォームのみ拡張を作成 |
| `ProcessBufferDefragmentation(FRDGBuilder&)` | 断片化したバッファをコンパクション |
| `PostBuildNaniteShadingCommands(...)` | シェーディングコマンド確定後のバッファ更新 |
| `CreateHitProxyIDBuffer(FRDGBuilder&)` | エディタのヒットプロキシ用 ID バッファ作成 |
| `CreateDebugViewModeBuffer(FRDGBuilder&)` | デバッグビューモード用バッファ作成 |
