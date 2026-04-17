# A: Mesh SDF — オフラインビルド・Atlas 管理

- 対象: `Engine/Public/DistanceFieldAtlas.h`, `ScenePrivate.h:2070`
- 上位: [[10_distance_field_overview]]
- Reference: [[ref_df_atlas]]

---

## 概要

**Mesh SDF** はスタティックメッシュごとにオフラインでビルドされる Signed Distance Field。  
Atlas テクスチャにパックされて GPU に転送され、DFAO・DF Shadows・Lumen の共通基盤となる。

---

## FDistanceFieldVolumeData

`Engine/Source/Runtime/Engine/Public/DistanceFieldAtlas.h:240`

```cpp
class FDistanceFieldVolumeData : public FDeferredCleanupInterface
{
    FBox3f LocalSpaceMeshBounds;      // ローカル空間バウンド
    bool   bMostlyTwoSided;           // 両面マテリアル判定フラグ
    bool   bAsyncBuilding;            // 非同期ビルド中フラグ

    // Mip 階層（NumMips 段）
    TStaticArray<FSparseDistanceFieldMip, DistanceField::NumMips> Mips;

    TArray<uint8>  AlwaysLoadedMip;   // 最低解像度 Mip（常にメモリ常駐）
    FByteBulkData  StreamableMips;    // 高解像度 Mip（ストリーミング）

    uint64         Id;                // アセット識別 ID
    FName          AssetName;         // デバッグ・統計用
};
```

| メンバ | 説明 |
|--------|------|
| `LocalSpaceMeshBounds` | SDF ボリュームのローカル空間 AABB |
| `bMostlyTwoSided` | 大部分が両面マテリアル → SDF 符号が反転する場合あり |
| `Mips` | スパース表現の Mip チェーン（未使用ボクセルをスキップ） |
| `AlwaysLoadedMip` | 最低解像度は必ず保持（カリング用途に使用） |
| `StreamableMips` | 高解像度 Mip はストリーミング（BulkData） |

---

## オフラインビルドフロー

```
UStaticMesh::CacheDerivedData()  [エディタ / Cook]
  │
  └─ FDistanceFieldVolumeData::CacheDerivedData(Mesh, GenerateSource, ...)
       │
       ├─ DDC（Derived Data Cache）を確認
       │   ヒット → 既存データをロード
       │
       └─ ミス → FDistanceFieldAsyncQueue::AddTask()
                  └─ FAsyncDistanceFieldTaskWorker::DoWork()  [非同期スレッド]
                       └─ GenerateSignedDistanceFieldVolumeData()
                            ├─ メッシュを UE-SDF 形式にボクセル化
                            ├─ 距離値を各 Mip に格納
                            └─ DDC に保存
```

**主要 CVar:**
- `r.DistanceFields.MaxPerMeshResolution` — メッシュあたりの最大 SDF 解像度
- `r.DistanceFields.DefaultVoxelDensity` — デフォルトボクセル密度

---

## FDistanceFieldSceneData（ランタイム管理）

`Engine/Source/Runtime/Renderer/Private/ScenePrivate.h:2070`

シーンに存在する全メッシュ SDF の GPU バッファを管理するクラス。

```cpp
class FDistanceFieldSceneData
{
    void AddPrimitive(FPrimitiveSceneInfo*);    // プリミティブ登録
    void UpdatePrimitive(FPrimitiveSceneInfo*); // トランスフォーム更新
    void RemovePrimitive(FPrimitiveSceneInfo*); // 登録解除

    // GPU バッファ同期
    void UpdateDistanceFieldObjectBuffers(GraphBuilder, ...);
    void UpdateDistanceFieldAtlas(GraphBuilder, ...);

    bool HasPendingUploads() const;    // アップロード待ちあり
    bool HasPendingOperations() const; // Add/Update/Remove 待ちあり
};
```

---

## Atlas へのパッキング

Mesh SDF は **3D テクスチャアトラス** に詰め込まれ GPU から参照される。

```
Atlas Texture（R16F または R8）
┌────────────────────────────────┐
│  [MeshA Mip0] [MeshB Mip0] ...│
│  [MeshA Mip1] [MeshB Mip1] ...│
│  ...                          │
└────────────────────────────────┘
         ↑
FDistanceFieldSceneData が各メッシュの
AtlasAllocation（UVW オフセット・スケール）を管理
```

- DFAO・DFShadows は `Object SDF バッファ` 経由でアトラスをサンプリング
- Lumen は直接 Mesh SDF Atlas を参照してトレース

---

## コード実行フロー（ランタイム）

```
FScene::AddPrimitive()  [RenderThread]
  └─ FDistanceFieldSceneData::AddPrimitive()
       → PendingAddOperations に追加

毎フレーム:
  UpdateDistanceFieldAtlas(GraphBuilder, ExternalAccessQueue, ...)
    ├─ PendingAddOperations を処理
    │   ├─ Atlas への空き領域確保
    │   └─ ScatterUpload CS で GPU に転送
    ├─ PendingRemoveOperations を処理（領域解放）
    └─ IndicesToUpdateInObjectBuffers を元に Object バッファ更新

  UpdateDistanceFieldObjectBuffers(GraphBuilder, ...)
    → GPU 上の Object SDF バッファ（位置・スケール・アトラス UV）を更新
```

---

## 各システムとの参照方式

| 利用システム | 参照方法 |
|------------|---------|
| DFAO | Global SDF 経由（Object SDF は使用しない） |
| DF Shadows | Object SDF バッファ + アトラス直接サンプリング |
| Lumen | Mesh SDF Atlas + Global SDF の両方を使用 |
