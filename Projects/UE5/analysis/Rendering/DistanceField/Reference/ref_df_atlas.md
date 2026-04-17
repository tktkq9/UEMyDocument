# Ref: DistanceField Atlas — FDistanceFieldVolumeData / FDistanceFieldSceneData リファレンス

- 対象: `Engine/Public/DistanceFieldAtlas.h`, `Renderer/Private/ScenePrivate.h:2070`
- 上位: [[10_distance_field_overview]] | [[a_mesh_sdf]]

---

## FDistanceFieldVolumeData

`Engine/Source/Runtime/Engine/Public/DistanceFieldAtlas.h:240`

```cpp
class FDistanceFieldVolumeData : public FDeferredCleanupInterface
{
    FBox3f  LocalSpaceMeshBounds;   // ローカル空間 AABB
    bool    bMostlyTwoSided;        // 両面マテリアル判定（SDF 符号が反転する場合あり）
    bool    bAsyncBuilding;         // 非同期ビルド中フラグ

    // Mip 階層（NumMips 段のスパース表現）
    TStaticArray<FSparseDistanceFieldMip, DistanceField::NumMips> Mips;

    TArray<uint8>  AlwaysLoadedMip; // 最低解像度 Mip（常にメモリ常駐・カリング用）
    FByteBulkData  StreamableMips;  // 高解像度 Mip（オンデマンドストリーミング）

    uint64  Id;           // アセット識別 ID
    FName   AssetName;    // デバッグ・統計用名前
};
```

| メンバ | 型 | 説明 |
|--------|-----|------|
| `LocalSpaceMeshBounds` | `FBox3f` | SDF ボリュームのローカル AABB |
| `bMostlyTwoSided` | `bool` | 両面ポリゴン主体のメッシュ → 内外判定に影響 |
| `Mips` | `TStaticArray<FSparseDistanceFieldMip, NumMips>` | スパース Mip チェーン |
| `AlwaysLoadedMip` | `TArray<uint8>` | 最低解像度・常駐 Mip |
| `StreamableMips` | `FByteBulkData` | ストリーミング対象の高解像度 Mip |
| `Id` | `uint64` | アセット固有 ID |
| `AssetName` | `FName` | デバッグ表示用 |

---

## FDistanceFieldAsyncQueue

`Engine/Source/Runtime/Engine/Public/DistanceFieldAtlas.h:348`

非同期スレッドで Mesh SDF をビルドするキュー。

```cpp
class FDistanceFieldAsyncQueue
{
    void AddTask(FDistanceFieldVolumeData* VolumeData, ...);  // ビルドタスク追加
    void BlockUntilBuildComplete(UStaticMesh* StaticMesh, bool bWarnIfBlocked); // 完了待機
    void ProcessAsyncTasks(bool bLimitExecutionTime);         // タスク処理（毎フレーム呼ばれる）
    int32 GetNumOutstandingTasks() const;                     // 未完了タスク数
};

// グローバルシングルトン
extern ENGINE_API FDistanceFieldAsyncQueue* GDistanceFieldAsyncQueue;
```

**ビルドフロー：**
```
UStaticMesh::CacheDerivedData()
  └─ FDistanceFieldVolumeData::CacheDerivedData()
       ├─ DDC ヒット → ロード完了
       └─ DDC ミス  → GDistanceFieldAsyncQueue->AddTask()
                        └─ FAsyncDistanceFieldTaskWorker::DoWork()
                             └─ GenerateSignedDistanceFieldVolumeData()
```

---

## FDistanceFieldSceneData

`Engine/Source/Runtime/Renderer/Private/ScenePrivate.h:2070`

シーン全体のすべての Mesh SDF を管理する GPU バッファ管理クラス。

```cpp
class FDistanceFieldSceneData
{
    // プリミティブ操作（AddPrimitive/Update/Remove でキューに追加）
    void AddPrimitive(FPrimitiveSceneInfo*);
    void UpdatePrimitive(FPrimitiveSceneInfo*);
    void RemovePrimitive(FPrimitiveSceneInfo*);

    // GPU バッファ同期（毎フレーム RenderThread で呼ばれる）
    void UpdateDistanceFieldObjectBuffers(FRDGBuilder& GraphBuilder, ...);
    void UpdateDistanceFieldAtlas(FRDGBuilder& GraphBuilder, FRDGExternalAccessQueue&, ...);

    // 状態クエリ
    bool HasPendingUploads() const;    // アップロード待ちデータあり
    bool HasPendingOperations() const; // Add/Update/Remove 未処理あり
    int32 NumObjectsInBuffer;          // GPU 上の有効オブジェクト数
};
```

---

## UpdateDistanceFieldAtlas() フロー

```
UpdateDistanceFieldAtlas(GraphBuilder, ExternalAccessQueue, ...)
  │
  ├─ PendingAddOperations を処理
  │   ├─ Atlas へ空き領域確保（3D テクスチャアトラス）
  │   └─ ScatterUpload CS → Mesh SDF データを GPU アトラスへ転送
  │
  ├─ PendingRemoveOperations を処理
  │   └─ Atlas 領域解放（フリーリスト更新）
  │
  └─ PendingUpdateOperations を処理
      └─ UpdateDistanceFieldObjectBuffers → 位置・スケール・AtlasUV を更新
```

---

## Atlas テクスチャ構造

```
Atlas Texture（R16F または R8、3D テクスチャ）
┌────────────────────────────────────┐
│  [MeshA Mip0] [MeshB Mip0] ...    │
│  [MeshA Mip1] [MeshB Mip1] ...    │
│  ...                              │
└────────────────────────────────────┘
         ↑
各メッシュの AtlasAllocation（UVW オフセット + スケール）を
FDistanceFieldSceneData が管理
```

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.DistanceFields.MaxPerMeshResolution` | メッシュあたりの最大 SDF 解像度 |
| `r.DistanceFields.DefaultVoxelDensity` | デフォルトのボクセル密度 |
| `r.DistanceFields.AtlasAllocSize` | Atlas テクスチャのアロケーションサイズ |
