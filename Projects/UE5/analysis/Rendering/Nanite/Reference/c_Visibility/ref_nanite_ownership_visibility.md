# リファレンス：NaniteOwnershipVisibilitySceneExtension.h / .cpp

- グループ: c - Visibility
- 上位: [[c_nanite_visibility]]
- 関連: [[ref_nanite_visibility]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteOwnershipVisibilitySceneExtension.h/.cpp`

---

## 概要

`bOwnerNoSee`（オーナーには見えない）/ `bOnlyOwnerSee`（オーナーのみ見える）の  
アクター可視性制御を GPU 側で処理する `ISceneExtension` 実装。  
ビューごとのオーナーシップビットマスクバッファを生成し、  
カリングシェーダーがそれを参照してインスタンスを除外する。

---

## 主要クラス

```cpp
namespace Nanite
{

// ISceneExtension 実装
class FOwnershipVisibilitySceneExtension : public ISceneExtension
{
public:
    static bool ShouldCreateExtension(const FScene& Scene);

    // オーナーシップを持つプリミティブ一覧を取得
    bool GetPrimitivesWithOwnership(
        TArray<FPrimitiveSceneInfo*>& OutPrimitives) const;

    // バッファのインデックス上限（アロケーション管理用）
    uint32 GetMaxPersistentPrimitiveIndex() const;

    // --- ISceneExtension サブクラス ---
    class FUpdater : public ISceneExtensionUpdater
    {
        // プリミティブ追加時: bOwnerNoSee / bOnlyOwnerSee フラグを記録
        void PreSceneUpdate(FScenePreUpdateChangeSet&) override;
        // プリミティブ削除時: エントリを削除
        void PostSceneUpdate(FScenePostUpdateChangeSet&) override;
    };

    class FRenderer : public ISceneExtensionRenderer
    {
        // ビューごとのオーナーシップビットマスクバッファを RDG 上に生成
        // カリングシェーダーが参照する
        FRDGBufferRef GetOwnershipVisibilityBuffer(
            FRDGBuilder& GraphBuilder,
            const FSceneView& View) const;

        void PostRenderOpaque(FPostRenderOpaqueInputs&) override;
    };
};

} // namespace Nanite
```

---

## 主要関数

| 関数 | 説明 |
|------|------|
| `ShouldCreateExtension(FScene&)` | bOwnerNoSee / bOnlyOwnerSee を使うプリミティブがある場合のみ有効 |
| `GetPrimitivesWithOwnership(...)` | オーナーシップ対象プリミティブのリストを返す |
| `GetOwnershipVisibilityBuffer(FRDGBuilder&, FSceneView&)` | ビュー別の可視性ビットマスクバッファを返す |
