# リファレンス：LumenSceneData.h

- グループ: a - Surface Cache
- 上位: [[a_lumen_surface_cache]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneData.h`

---

## クラス・構造体一覧

| 名前 | 種別 | 役割 |
|------|------|------|
| `FLumenCardId` | union | Card の一意識別子（64bit パック）|
| `FLumenCardSharingInfo` | struct | Card 共有情報（インデックス・解像度・フリップ）|
| `FLumenSurfaceMipMap` | struct | Card の 1 ミップレベルのアトラス割り当て情報 |
| `FLumenMipMapDesc` | struct | ミップレベルの解像度・ページサイズ記述 |
| `FLumenCard` | class | 1 枚の平面パッチ（Surface Cache の基本単位）|
| `FLumenPrimitiveGroupRemoveInfo` | class | プリミティブ削除の遅延情報 |
| `FLumenPrimitiveGroup` | class | プリミティブのグループ（MeshCards に対応）|
| `FLumenPrimitiveGroupCullingInfo` | struct | GPU Driven カリング用のコンパクトな情報 |
| `FLumenPageTableEntry` | struct | 物理ページの割り当て情報（アトラス矩形・サンプリング情報）|
| `FSurfaceCacheRequest` | class | Surface Cache 更新リクエスト |
| `FVirtualPageIndex` | union | 仮想ページインデックス（64bit パック）|
| `FLumenSurfaceCacheAllocator` | class | 物理ページのビン方式アロケータ |
| `ESurfaceCacheCompression` | enum class | Surface Cache の圧縮方式 |
| `FLumenSharedRT` | class | RDG テクスチャのラッパー（共有 RT 管理）|
| `FLumenViewOrigin` | struct | Lumen シーン更新に使うビュー原点情報 |
| `FLumenSceneFrameTemporaries` | struct | 1 フレーム限定の一時リソース（全アトラス RDG テクスチャ）|
| `FLumenSceneData` | class | Lumen シーン全体の CPU 側メインデータ |

---

## FLumenCardId

```cpp
union FLumenCardId {
    uint64 PackedValue;
    struct {
        uint32 ResLevelBiasX         : 4;  // X 方向解像度バイアス
        uint32 ResLevelBiasY         : 4;  // Y 方向解像度バイアス
        uint32 AxisAlignedDirectionIndex : 3; // 方向インデックス（0〜5）
        uint32 Unused                : 21;
        uint32 CustomId;                   // メッシュ固有 ID
    };
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `PackedValue` | `uint64` | 全フィールドを詰めた 64bit 値 |
| `ResLevelBiasX` | `uint32 : 4` | X 方向の解像度バイアス（0〜15）|
| `ResLevelBiasY` | `uint32 : 4` | Y 方向の解像度バイアス（0〜15）|
| `AxisAlignedDirectionIndex` | `uint32 : 3` | 方向インデックス（0=+X, 1=-X, 2=+Y, 3=-Y, 4=+Z, 5=-Z）|
| `CustomId` | `uint32` | メッシュ固有の ID（Card 共有の照合キー）|

### 主要メソッド

| メソッド | 説明 |
|---------|------|
| `FLumenCardId(uint32 InCustomId, uint8 InAxisAlignedDirectionIndex, uint8 InResLevelBiasX, uint8 InResLevelBiasY)` | 各フィールドを指定して構築 |
| `bool IsValid() const` | PackedValue が無効値（全ビット1）でなければ true |
| `void Invalidate()` | 無効値（PackedValue = MAX_uint64）にセット |
| `static constexpr FLumenCardId GetInvalidId()` | 無効 ID を返す |
| `bool operator<(const FLumenCardId&) const` | PackedValue による比較（TMap キー用）|

### 使用箇所
- [[ref_lumen_scene_data]] `FLumenCard::CardSharingId` — Card 共有の照合キーとして保持
- [[ref_lumen_scene_data]] `FLumenSceneData::CardSharingInfoMap` — 共有 Card のルックアップマップのキー

---

## FLumenSurfaceMipMap

Card の 1 ミップレベルに対するアトラス割り当て状態を管理する構造体。

```cpp
struct FLumenSurfaceMipMap {
    uint8 SizeInPagesX;        // X 方向のページ数（0 = 未割り当て）
    uint8 SizeInPagesY;        // Y 方向のページ数
    uint8 ResLevelX;           // X 方向の解像度レベル
    uint8 ResLevelY;           // Y 方向の解像度レベル
    int32 PageTableSpanOffset; // ページテーブル上の開始オフセット（-1 = 未割り当て）
    uint16 PageTableSpanSize;  // ページテーブルの占有数
    bool bLocked;              // メモリに固定（アンロードされない）か
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `SizeInPagesX/Y` | `uint8` | 各軸のページ数（SizeInPages = テクセル数 / PhysicalPageSize）|
| `ResLevelX/Y` | `uint8` | 解像度レベル（`Lumen::MinResLevel` 〜 `Lumen::MaxResLevel`）|
| `PageTableSpanOffset` | `int32` | `FLumenSceneData::PageTable` の先頭インデックス（-1 = 未割り当て）|
| `PageTableSpanSize` | `uint16` | ページテーブル上の連続エントリ数（= SizeInPagesX * SizeInPagesY）|
| `bLocked` | `bool` | true のとき LRU ヒープから除外され強制的に保持される |

### メソッド

| メソッド | 説明 |
|---------|------|
| `bool IsAllocated() const` | `PageTableSpanSize > 0` のとき true |
| `FIntPoint GetSizeInPages() const` | `{SizeInPagesX, SizeInPagesY}` を返す |
| `int32 GetPageTableIndex(int32 LocalPageIndex) const` | ローカルページインデックス → グローバルページテーブルインデックス変換 |

---

## FLumenCard

> **概要**: Surface Cache の基本単位。メッシュ表面の 1 方向の平面パッチ。OBB と物理ページ割り当てを保持する。

```cpp
class FLumenCard {
public:
    FLumenCardOBBf LocalOBB;      // ローカル空間 OBB
    FLumenCardOBBd WorldOBB;      // ワールド空間 OBB（double 精度）
    FLumenCardOBBf MeshCardsOBB;  // MeshCards 空間の OBB

    bool bVisible;
    bool bHeightfield;
    bool bAxisXFlipped;
    ELumenCardDilationMode DilationMode;

    uint8 MinAllocatedResLevel;
    uint8 MaxAllocatedResLevel;
    uint8 DesiredLockedResLevel;

    FLumenSurfaceMipMap SurfaceMipMaps[Lumen::NumResLevels]; // 9 段分

    int32 MeshCardsIndex;
    int32 IndexInMeshCards;
    uint8 IndexInBuildData;
    uint8 AxisAlignedDirectionIndex;
    float ResolutionScale;
    float CardAspect;

    FLumenCardId CardSharingId;
    int32 CardSharingListIndex;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `LocalOBB` | `FLumenCardOBBf` | float 精度のローカル空間 OBB |
| `WorldOBB` | `FLumenCardOBBd` | double 精度のワールド空間 OBB（大規模ワールド対応）|
| `MeshCardsOBB` | `FLumenCardOBBf` | MeshCards ローカル空間での OBB（相対変換用）|
| `bVisible` | `bool` | シーン内で可視と判定されているか |
| `bHeightfield` | `bool` | ランドスケープ用ハイトフィールド Card か |
| `bAxisXFlipped` | `bool` | X 軸が反転しているか（カリング判定に使用）|
| `DilationMode` | `ELumenCardDilationMode` | Card 端のディレーション方式（エッジアーティファクト抑制）|
| `MinAllocatedResLevel` | `uint8` | 割り当て済みミップの最小解像度レベル（UINT8_MAX = 未割り当て）|
| `MaxAllocatedResLevel` | `uint8` | 割り当て済みミップの最大解像度レベル |
| `DesiredLockedResLevel` | `uint8` | カメラ距離から算出した希望解像度レベル |
| `SurfaceMipMaps` | `FLumenSurfaceMipMap[NumResLevels]` | 解像度レベルごとのページ割り当て情報（最大 9 段）|
| `MeshCardsIndex` | `int32` | 親 `FLumenMeshCards` のインデックス |
| `IndexInMeshCards` | `int32` | MeshCards 内での Card の位置（0〜5）|
| `IndexInBuildData` | `uint8` | `FMeshCardsBuildData` 内のインデックス |
| `AxisAlignedDirectionIndex` | `uint8` | 投影方向（0=+X, 1=-X, 2=+Y, 3=-Y, 4=+Z, 5=-Z）|
| `ResolutionScale` | `float` | 解像度カスタムスケール（デフォルト 1.0）|
| `CardAspect` | `float` | Card の縦横比（初期化後変更不可）|
| `CardSharingId` | `FLumenCardId` | Card 共有の照合キー |
| `CardSharingListIndex` | `int32` | `CardSharingInfoMap` 内の共有エントリインデックス（-1 = 非共有）|

---

## FLumenCard::Initialize

```cpp
void Initialize(
    float InResolutionScale,
    uint32 CustomId,
    const FMatrix& LocalToWorld,
    const FLumenMeshCards& InMeshCardsInstance,
    const FLumenCardBuildData& CardBuildData,
    int32 InIndexInMeshCards,
    int32 InMeshCardsIndex,
    uint8 InIndexInBuildData);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `InResolutionScale` | `float` | 解像度スケール乗数（`CardResolutionScale` から引き継ぐ）|
| `CustomId` | `uint32` | Card 共有照合用のメッシュ固有 ID |
| `LocalToWorld` | `const FMatrix&` | 初期ローカル→ワールド変換行列 |
| `InMeshCardsInstance` | `const FLumenMeshCards&` | 親 MeshCards インスタンス |
| `CardBuildData` | `const FLumenCardBuildData&` | ビルド時に計算された Card の OBB・方向情報 |
| `InIndexInMeshCards` | `int32` | この Card の MeshCards 内での位置 |
| `InMeshCardsIndex` | `int32` | 親 MeshCards のシーンインデックス |
| `InIndexInBuildData` | `uint8` | ビルドデータ配列内のインデックス |

### 使用箇所
- [[ref_lumen_scene]] `FLumenSceneData::AddMeshCardsFromBuildData()` — Card スパン確保後に各 Card を初期化するために呼ばれる

### 内部処理フロー

1. **OBB・インデックスのコピー**
   ```cpp
   LocalOBB = CardBuildData.OBB;
   AxisAlignedDirectionIndex = CardBuildData.AxisAlignedDirectionIndex;
   IndexInMeshCards = InIndexInMeshCards;
   MeshCardsIndex   = InMeshCardsIndex;
   IndexInBuildData = InIndexInBuildData;
   ```

2. **カード共有 ID の設定**
   ```cpp
   CardSharingId = FLumenCardId(CustomId, AxisAlignedDirectionIndex,
       ResLevelToResLevelXYBias().X, ResLevelToResLevelXYBias().Y);
   ```

3. **ワールド変換の適用**
   ```cpp
   SetTransform(LocalToWorld, InMeshCardsInstance);
   ```

4. **ミップマップの初期化**
   ```cpp
   for (auto& MipMap : SurfaceMipMaps) {
       MipMap.PageTableSpanOffset = -1;
       MipMap.PageTableSpanSize   = 0;
   }
   MinAllocatedResLevel = UINT8_MAX;
   MaxAllocatedResLevel = 0;
   ```

---

## FLumenCard::SetTransform

```cpp
void SetTransform(const FMatrix& LocalToWorld, const FLumenMeshCards& MeshCards);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `LocalToWorld` | `const FMatrix&` | 新しいローカル→ワールド変換 |
| `MeshCards` | `const FLumenMeshCards&` | 親 MeshCards（WorldToLocalRotation を使用）|

### 使用箇所
- [[ref_lumen_scene_data]] `FLumenCard::Initialize()` — 初期化時
- [[ref_lumen_scene]] `FLumenSceneData::UpdateMeshCards()` — トランスフォーム変化時に各 Card に対して呼ばれる

### 内部処理フロー

1. **WorldOBB の計算**
   ```cpp
   WorldOBB = LocalOBB.TransformBy(LocalToWorld); // double 精度で保持
   ```
2. **MeshCardsOBB の計算**（MeshCards ローカル空間への変換）
   ```cpp
   MeshCardsOBB = LocalOBB.TransformBy(MeshCards.WorldToLocalRotation * LocalToWorld);
   ```

---

> [!note]- FLumenCard::UpdateMinMaxAllocatedLevel — 割り当て済み解像度範囲を再計算
> 
> ```cpp
> void UpdateMinMaxAllocatedLevel();
> ```
> 
> **説明**: 全 `SurfaceMipMaps` を走査し、`IsAllocated()` が true のミップレベルの最小・最大を `MinAllocatedResLevel` / `MaxAllocatedResLevel` に書き込む。
> 
> **使用箇所**
> - [[ref_lumen_scene_data]] `FLumenSceneData::FreeVirtualSurface()` — ページ解放後に範囲を更新
> - [[ref_lumen_scene_data]] `FLumenSceneData::ReallocVirtualSurface()` — ページ確保後に範囲を更新

> [!note]- FLumenCard::GetMipMap / GetMipMapDesc — ミップマップ情報取得
> 
> ```cpp
> FLumenSurfaceMipMap& GetMipMap(int32 ResLevel);
> void GetMipMapDesc(int32 ResLevel, FLumenMipMapDesc& OutDesc) const;
> ```
> 
> **パラメータ**
> 
> | 引数 | 型 | 説明 |
> |------|-----|------|
> | `ResLevel` | `int32` | 解像度レベル（`Lumen::MinResLevel` 〜 `Lumen::MaxResLevel`）|
> | `OutDesc` | `FLumenMipMapDesc&` | 出力: テクセル解像度・ページサイズ情報 |
> 
> **使用箇所**
> - [[ref_lumen_scene_data]] `FLumenSceneData::ReallocVirtualSurface()` — ページ確保対象の MipMap を取得
> - [[ref_lumen_surface_cache]] `BuildCardUpdateList()` — テクセル数計算に使用

> [!note]- FLumenCard::GetSurfaceStats — テクセル使用統計
> 
> ```cpp
> void GetSurfaceStats(
>     const TSparseSpanArray<FLumenPageTableEntry>& PageTable,
>     FSurfaceStats& OutStats) const;
> ```
> 
> **使用箇所**: デバッグ表示（`stat lumen`）でアトラス使用量を集計する際に呼ばれる

---

## FLumenPrimitiveGroup

> **概要**: 1 プリミティブ（またはインスタンスグループ）に対応する Lumen 管理単位。MeshCards への橋渡しを担う。

```cpp
class FLumenPrimitiveGroup {
public:
    TArray<FPrimitiveSceneInfo*, TInlineAllocator<1>> Primitives;
    int32 PrimitiveInstanceIndex;
    int32 MeshCardsIndex;
    int32 HeightfieldIndex;
    int32 PrimitiveCullingInfoIndex;
    int32 InstanceCullingInfoIndex;
    uint32 CustomId;
    Experimental::FHashElementId RayTracingGroupMapElementId;
    float CardResolutionScale;

    uint8 bValidMeshCards      : 1;
    uint8 bFarField            : 1;
    uint8 bHeightfield         : 1;
    uint8 bEmissiveLightSource : 1;
    uint8 bOpaqueOrMasked      : 1;
    uint8 bLandscape           : 1;
    uint8 LightingChannelMask;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Primitives` | `TArray<FPrimitiveSceneInfo*, TInlineAllocator<1>>` | このグループに属するプリミティブ（通常 1 つ、マージ時は複数）|
| `PrimitiveInstanceIndex` | `int32` | インスタンス化時の個別インスタンスインデックス（-1 = 非インスタンス）|
| `MeshCardsIndex` | `int32` | 対応する `FLumenMeshCards` のインデックス（-1 = 未割り当て）|
| `HeightfieldIndex` | `int32` | ハイトフィールドインデックス（-1 = 非ハイトフィールド）|
| `PrimitiveCullingInfoIndex` | `int32` | GPU Driven カリング情報のインデックス（INDEX_NONE = なし）|
| `InstanceCullingInfoIndex` | `int32` | インスタンス単位カリング情報インデックス |
| `CustomId` | `uint32` | Card 共有の照合用 ID（メッシュの同一性確認）|
| `RayTracingGroupMapElementId` | `FHashElementId` | Ray Tracing グループマップ内のエントリ ID |
| `CardResolutionScale` | `float` | Card 解像度スケール（プリミティブの `r.Lumen.Detail` 設定由来）|
| `bValidMeshCards` | `uint8 : 1` | MeshCards が有効（Card が 1 枚以上生成された）か |
| `bFarField` | `uint8 : 1` | 遠距離フィールドとして扱うか |
| `bHeightfield` | `uint8 : 1` | ランドスケープのハイトフィールドか |
| `bEmissiveLightSource` | `uint8 : 1` | 自発光光源として GI に寄与するか |
| `bOpaqueOrMasked` | `uint8 : 1` | 不透明またはマスクマテリアルか（半透明は Lumen Card 対象外）|
| `bLandscape` | `uint8 : 1` | ランドスケーププロキシか |
| `LightingChannelMask` | `uint8` | ライティングチャンネルマスク（0〜7 ビット）|

### メソッド

| メソッド | 説明 |
|---------|------|
| `bool HasMergedInstances() const` | 複数インスタンスがこのグループにまとめられているか |
| `bool HasMergedPrimitives() const` | 複数プリミティブが Ray Tracing グループとしてまとめられているか |

### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::PrimitiveGroups` — シーン全体の PrimitiveGroup を管理する `TSparseSpanArray`
- [[ref_lumen_scene]] `UpdateLumenScenePrimitives()` — プリミティブ追加時にグループを作成
- [[ref_lumen_mesh_cards]] `FLumenMeshCards::PrimitiveGroupIndex` — 逆引きリンク

---

## FLumenPageTableEntry

> **概要**: 仮想ページ（Card のミップレベルの 1 ページ）から物理アトラス上の矩形へのマッピング情報。

```cpp
struct FLumenPageTableEntry {
    FIntPoint PhysicalPageCoord;   // 物理ページ座標（-1,-1 = 未割り当て）
    FIntRect  PhysicalAtlasRect;   // アトラス上の矩形（テクセル単位）

    uint32 SamplePageIndex;
    uint16 SampleAtlasBiasX, SampleAtlasBiasY;
    uint16 SampleCardResLevelX, SampleCardResLevelY;

    int32 CardIndex;
    uint8 ResLevel;
    FVector4f CardUVRect;
    FIntPoint SubAllocationSize;   // (-1,-1) = サブアロケーションなし
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `PhysicalPageCoord` | `FIntPoint` | 物理ページ配列上の座標（`(-1,-1)` = 未マップ）|
| `PhysicalAtlasRect` | `FIntRect` | アトラステクスチャ上の実際の矩形（テクセル単位）|
| `SamplePageIndex` | `uint32` | サンプリング時に参照するページインデックス（低解像度フォールバック用）|
| `SampleAtlasBiasX/Y` | `uint16` | サンプリングページへのアトラス座標オフセット |
| `SampleCardResLevelX/Y` | `uint16` | サンプリングページの解像度レベル |
| `CardIndex` | `int32` | このページが属する `FLumenCard` のインデックス |
| `ResLevel` | `uint8` | このページの解像度レベル |
| `CardUVRect` | `FVector4f` | Card 空間での UV 矩形（シェーダーで直接使用）|
| `SubAllocationSize` | `FIntPoint` | サブアロケーションのサイズ（`(-1,-1)` = 物理 1 ページ丸ごと使用）|

### メソッド

| メソッド | 説明 |
|---------|------|
| `bool IsSubAllocation() const` | `SubAllocationSize != (-1,-1)` のとき true |
| `bool IsMapped() const` | `PhysicalPageCoord != (-1,-1)` のとき true |
| `uint32 GetNumVirtualTexels() const` | 仮想テクセル数（= CardUVRect から計算）|
| `uint32 GetNumPhysicalTexels() const` | 物理テクセル数（= PhysicalAtlasRect の面積）|

### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::PageTable` — シーン全体の仮想ページテーブル (`TSparseSpanArray`)
- [[ref_lumen_surface_cache]] `FLumenSurfaceCacheAllocator::Allocate()` — 物理ページ確保後にフィールドを書き込む
- [[ref_lumen_tracing_utils]] `FLumenCardTracingParameters` — `PageTableBufferSRV` としてシェーダーに渡される

---

## FLumenSurfaceCacheAllocator

> **概要**: 物理ページのビン方式アロケータ。小サイズ要素はサブアロケーションで 1 物理ページ内に詰め込む。

```cpp
class FLumenSurfaceCacheAllocator {
public:
    struct FAllocation {
        FIntPoint PhysicalPageCoord;
        FIntRect  PhysicalAtlasRect;
    };

    void Init(const FIntPoint& InPageAtlasSizeInPages);
    void Allocate(const FLumenPageTableEntry& Page, FAllocation& OutAllocation);
    void Free(const FLumenPageTableEntry& Page);
    bool IsSpaceAvailable(const FLumenCard& Card, int32 ResLevel, bool bSinglePage) const;
    void GetStats(FStats& OutStats) const;
};
```

### Init

```cpp
void Init(const FIntPoint& InPageAtlasSizeInPages);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `InPageAtlasSizeInPages` | `const FIntPoint&` | 物理アトラスの縦横ページ数（例: `{32, 32}` = 4096px / 128px）|

#### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::AllocateCardAtlases()` — アトラスサイズ確定後に呼ばれる

---

### Allocate

```cpp
void Allocate(const FLumenPageTableEntry& Page, FAllocation& OutAllocation);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `Page` | `const FLumenPageTableEntry&` | 確保対象ページ（`SubAllocationSize` と `ResLevel` を参照）|
| `OutAllocation` | `FAllocation&` | 出力: 確保した物理ページ座標とアトラス矩形 |

#### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::ReallocVirtualSurface()` — 仮想サーフェス確保時に呼ばれる

#### 内部処理フロー

1. **サブアロケーションサイズの決定**
   ```cpp
   if (Page.IsSubAllocation()) {
       BinIndex = PageBinLookup[log2(Page.SubAllocationSize.X)][log2(Page.SubAllocationSize.Y)];
   } else {
       // 物理ページ 1 枚丸ごと確保
   }
   ```

2. **フリーページビンから確保**
   ```cpp
   FPageBin& Bin = PageBins[BinIndex];
   if (Bin.FreeList.IsEmpty()) {
       // 新しい物理ページを AllocatePhysicalAtlasPage() で確保
       Bin.FreeList.Add(NewPage);
   }
   OutAllocation.PhysicalPageCoord = Bin.FreeList.Pop();
   ```

3. **アトラス矩形の計算**
   ```cpp
   OutAllocation.PhysicalAtlasRect = FIntRect(
       PhysicalPageCoord * PhysicalPageSize,
       PhysicalPageCoord * PhysicalPageSize + SubAllocationSize);
   ```

---

### Free

```cpp
void Free(const FLumenPageTableEntry& Page);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `Page` | `const FLumenPageTableEntry&` | 解放対象ページ（`PhysicalPageCoord` と `SubAllocationSize` を参照）|

#### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::FreeVirtualSurface()` — ページアンロード時
- [[ref_lumen_scene_data]] `FLumenSceneData::RemoveCardFromAtlas()` — Card 削除時に全ページを解放

---

> [!note]- IsSpaceAvailable — 空き容量チェック
> 
> ```cpp
> bool IsSpaceAvailable(const FLumenCard& Card, int32 ResLevel, bool bSinglePage) const;
> ```
> 
> **パラメータ**
> 
> | 引数 | 型 | 説明 |
> |------|-----|------|
> | `Card` | `const FLumenCard&` | 確保を試みる Card |
> | `ResLevel` | `int32` | 確保したい解像度レベル |
> | `bSinglePage` | `bool` | true = 単一物理ページに収まるか確認、false = サブアロケーション可能か確認 |
> 
> **使用箇所**: [[ref_lumen_surface_cache]] `UpdateSurfaceCacheAllocationState()` — 解像度を上げるべきか判断する前に空き容量を確認

---

## FLumenViewOrigin

> **概要**: Lumen シーン更新（Card 解像度決定・トレース距離計算）に使うビュー情報をコンパクトにまとめた構造体。複数ビュー（キューブマップ・nDisplay 等）への対応を可能にする。

```cpp
struct FLumenViewOrigin {
    const FSceneViewFamily* Family;
    FVector  LumenSceneViewOrigin;
    FVector4f WorldCameraOrigin;
    FDFVector3 PreViewTranslationDF;
    FMatrix44f FrustumTranslatedWorldToClip;
    FMatrix44f ViewToClip;
    float OrthoMaxDimension;
    float LastEyeAdaptationExposure;
    float MaxTraceDistance;
    float CardMaxDistance;
    float LumenSceneDetail;
    const FViewInfo* ReferenceView;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Family` | `const FSceneViewFamily*` | 参照元のビューファミリ（ShowFlags 参照用）|
| `LumenSceneViewOrigin` | `FVector` | Lumen シーン座標系でのビュー原点（大規模ワールド対応のためオフセット済み）|
| `WorldCameraOrigin` | `FVector4f` | ワールド空間カメラ原点（シェーダー渡し用 float4）|
| `PreViewTranslationDF` | `FDFVector3` | ビュー前変換（double-float 形式、プレシジョン対策）|
| `FrustumTranslatedWorldToClip` | `FMatrix44f` | 視錐台クリップ変換行列（float 精度）|
| `ViewToClip` | `FMatrix44f` | ビュー→クリップ変換行列 |
| `OrthoMaxDimension` | `float` | 正投影の最大次元（0 = 透視投影）|
| `LastEyeAdaptationExposure` | `float` | 前フレームの露出値（Emissive スケール用）|
| `MaxTraceDistance` | `float` | Lumen の最大レイトレース距離（`r.Lumen.MaxTraceDistance` 由来）|
| `CardMaxDistance` | `float` | Card がキャプチャされる最大距離（`LumenScene::GetCardMaxDistance()` 由来）|
| `LumenSceneDetail` | `float` | シーン詳細度（`r.Lumen.Scene.Detail` 由来）|
| `ReferenceView` | `const FViewInfo*` | 元となる `FViewInfo`（後方互換のために保持）|

### メソッド

## FLumenViewOrigin::Init

```cpp
void Init(const FViewInfo& View);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `View` | `const FViewInfo&` | 初期化元のビュー情報 |

#### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneFrameTemporaries::ViewOrigins` の各要素を初期化する際に呼ばれる
- [[ref_lumen_scene]] `UpdateLumenScene()` — フレーム開始時に各ビューから ViewOrigin を生成

#### 内部処理フロー

1. **座標系の設定**
   ```cpp
   LumenSceneViewOrigin = View.ViewMatrices.GetViewOrigin();
   WorldCameraOrigin    = FVector4f(View.ViewMatrices.GetViewOrigin(), 1.0f);
   PreViewTranslationDF = View.ViewMatrices.GetPreViewTranslation();
   ```

2. **行列のコピー**
   ```cpp
   FrustumTranslatedWorldToClip = FMatrix44f(View.ViewMatrices.GetTranslatedViewProjectionMatrix());
   ViewToClip = FMatrix44f(View.ViewMatrices.GetProjectionMatrix());
   ```

3. **距離パラメータの計算**
   ```cpp
   MaxTraceDistance = Lumen::GetMaxTraceDistance(View);
   CardMaxDistance  = LumenScene::GetCardMaxDistance(View);
   OrthoMaxDimension = View.IsPerspectiveProjection() ? 0.0f : View.ViewMatrices.GetOrthoDimension();
   ```

> [!note]- IsPerspectiveProjection
> 
> ```cpp
> bool IsPerspectiveProjection() const { return OrthoMaxDimension == 0.0f; }
> ```
> 
> `OrthoMaxDimension` が 0 かどうかで透視投影 / 正投影を判定する。

---

## FLumenSceneFrameTemporaries

> **概要**: 1 フレームの Lumen レンダリング中にのみ有効な RDG テクスチャ・バッファ・タスクを束ねた構造体。`FDeferredShadingRenderer` が 1 つ保持し、各サブシステムに渡す。

```cpp
struct FLumenSceneFrameTemporaries {
    // Surface Cache アトラス（書き込み用 RDG テクスチャ）
    FRDGTextureRef AlbedoAtlas;
    FRDGTextureRef OpacityAtlas;
    FRDGTextureRef NormalAtlas;
    FRDGTextureRef EmissiveAtlas;
    FRDGTextureRef DepthAtlas;
    FRDGTextureRef DirectLightingAtlas;
    FRDGTextureRef IndirectLightingAtlas;
    FRDGTextureRef RadiosityNumFramesAccumulatedAtlas;
    FRDGTextureRef FinalLightingAtlas;
    FRDGBufferRef  TileShadowDownsampleFactorAtlas;
    FRDGTextureRef DiffuseLightingAndSecondMomentHistoryAtlas;
    FRDGTextureRef NumFramesAccumulatedHistoryAtlas;

    // GPU バッファ SRV / UAV
    FRDGBufferSRV* CardBufferSRV;
    FRDGBufferSRV* MeshCardsBufferSRV;
    FRDGBufferSRV* HeightfieldBufferSRV;
    FRDGBufferSRV* PrimitiveGroupBufferSRV;
    FRDGBufferSRV* PageTableBufferSRV;
    FRDGBufferSRV* CardPageBufferSRV;
    FRDGBufferUAV* CardPageBufferUAV;
    FRDGBufferUAV* CardPageLastUsedBufferUAV;

    // Uniform Buffer
    TRDGUniformBufferRef<FLumenCardScene> LumenCardSceneUniformBuffer;

    // 非同期更新タスク
    UE::Tasks::FTask UpdateSceneTask;

    // フラグ・ビュー情報
    bool bReallocateAtlas;
    TArray<FLumenViewOrigin, TFixedAllocator<LUMEN_MAX_VIEWS>> ViewOrigins;

    // 反射・拡散 GI 共有 RT
    TArray<FLumenSharedRT> ReflectSpecularIndirect[(uint32)ELumenReflectionPass::MAX];

    // Surface Cache フィードバックリソース
    FLumenSurfaceCacheFeedback::FFeedbackResources SurfaceCacheFeedbackResources;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `AlbedoAtlas` | `FRDGTextureRef` | アルベドアトラス（PF_R8G8B8A8 または BC7 圧縮）|
| `OpacityAtlas` | `FRDGTextureRef` | 不透明度アトラス（PF_G8）|
| `NormalAtlas` | `FRDGTextureRef` | 法線アトラス（PF_R8G8 または BC5）|
| `EmissiveAtlas` | `FRDGTextureRef` | 自発光アトラス（PF_FloatR11G11B10 または BC6H）|
| `DepthAtlas` | `FRDGTextureRef` | 深度アトラス（PF_G16）|
| `DirectLightingAtlas` | `FRDGTextureRef` | 直接光ライティング結果アトラス |
| `IndirectLightingAtlas` | `FRDGTextureRef` | 間接光（Radiosity）結果アトラス |
| `RadiosityNumFramesAccumulatedAtlas` | `FRDGTextureRef` | Radiosity 蓄積フレーム数アトラス |
| `FinalLightingAtlas` | `FRDGTextureRef` | 最終ライティング合成結果アトラス |
| `TileShadowDownsampleFactorAtlas` | `FRDGBufferRef` | タイルごとのシャドウダウンサンプル係数バッファ |
| `DiffuseLightingAndSecondMomentHistoryAtlas` | `FRDGTextureRef` | 拡散 GI のテンポラル蓄積ヒストリ |
| `NumFramesAccumulatedHistoryAtlas` | `FRDGTextureRef` | テンポラル蓄積フレーム数ヒストリ |
| `CardBufferSRV` | `FRDGBufferSRV*` | Card データ GPU バッファの SRV（シェーダーで Card 情報を参照）|
| `MeshCardsBufferSRV` | `FRDGBufferSRV*` | MeshCards データ GPU バッファの SRV |
| `HeightfieldBufferSRV` | `FRDGBufferSRV*` | ハイトフィールドデータ GPU バッファの SRV |
| `PrimitiveGroupBufferSRV` | `FRDGBufferSRV*` | PrimitiveGroup データ GPU バッファの SRV |
| `PageTableBufferSRV` | `FRDGBufferSRV*` | ページテーブル GPU バッファの SRV |
| `CardPageBufferSRV` | `FRDGBufferSRV*` | Card ページリスト GPU バッファの SRV |
| `CardPageBufferUAV` | `FRDGBufferUAV*` | Card ページリスト GPU バッファの UAV（更新用）|
| `CardPageLastUsedBufferUAV` | `FRDGBufferUAV*` | Card ページ最終使用フレーム番号バッファの UAV |
| `LumenCardSceneUniformBuffer` | `TRDGUniformBufferRef<FLumenCardScene>` | 全アトラスと GPU バッファをまとめた Uniform Buffer（シェーダーバインド用）|
| `UpdateSceneTask` | `UE::Tasks::FTask` | Lumen シーン CPU 更新タスクハンドル（非同期実行）|
| `bReallocateAtlas` | `bool` | アトラスサイズ変更が必要か（`true` でフレーム開始時に再生成）|
| `ViewOrigins` | `TArray<FLumenViewOrigin, TFixedAllocator<LUMEN_MAX_VIEWS>>` | 各ビューの原点情報（最大 `LUMEN_MAX_VIEWS` 個）|
| `SurfaceCacheFeedbackResources` | `FLumenSurfaceCacheFeedback::FFeedbackResources` | フィードバックバッファの RDG UAV/SRV |

### 使用箇所
- [[ref_lumen_scene]] `UpdateLumenScene()` — 毎フレームここに全アトラスを登録して後続パスに渡す
- [[ref_lumen_scene_lighting]] `RenderLumenSceneLighting()` — アトラス UAV に直接光・Radiosity を書き込む
- [[ref_lumen_diffuse_indirect]] `RenderLumenDiffuseIndirect()` — `LumenCardSceneUniformBuffer` をバインドして GI トレースを行う
- [[ref_lumen_reflections]] `RenderLumenReflections()` — 同上

---

## FLumenSceneData

> **概要**: Lumen シーン全体の CPU 側メインデータを保持するクラス。Card・MeshCards・PrimitiveGroup の疎配列、GPU バッファ、Surface Cache アロケータ、保留中の追加/削除操作をすべて管理する。

### メンバ変数（主要）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Cards` | `TSparseSpanArray<FLumenCard>` | シーン全 Card の疎配列（安定インデックス）|
| `CardIndicesToUpdateInBuffer` | `FUniqueIndexList` | 次フレームで GPU バッファへアップロードが必要な Card インデックス |
| `CardBuffer` | `TRefCountPtr<FRDGPooledBuffer>` | Card データ GPU バッファ（`FLumenCardGPUData` の構造体配列）|
| `MeshCards` | `TSparseSpanArray<FLumenMeshCards>` | シーン全 MeshCards の疎配列 |
| `MeshCardsIndicesToUpdateInBuffer` | `FUniqueIndexList` | GPU アップロード待ちの MeshCards インデックス |
| `PrimitiveGroups` | `TSparseSpanArray<FLumenPrimitiveGroup>` | シーン全 PrimitiveGroup の疎配列 |
| `Heightfields` | `TSparseSpanArray<FLumenHeightfield>` | ハイトフィールドの疎配列 |
| `PageTable` | `TSparseSpanArray<FLumenPageTableEntry>` | 仮想ページテーブル全体 |
| `SurfaceCacheAllocator` | `FLumenSurfaceCacheAllocator` | 物理ページアロケータ |
| `UnlockedAllocationHeap` | `FBinaryHeap<uint32, uint32>` | LRU ヒープ（最も古く使われていないページを evict するための優先度キュー）|
| `PagesToRecaptureHeap` | `FBinaryHeap<uint32, uint32>[MAX_NUM_GPUS]` | 再キャプチャが必要なページの優先度キュー（GPU マスクごと）|
| `AlbedoAtlas` / ... | `TRefCountPtr<IPooledRenderTarget>` | 永続的なアトラステクスチャ（フレームをまたいで保持）|
| `SurfaceCacheResolution` | `float` | アトラスサイズスケール（`r.LumenScene.SurfaceCache.AtlasSize` 由来）|
| `PendingAddOperations` | `TSet<FPrimitiveSceneInfo*>` | 追加待ちプリミティブ（ゲームスレッドから積まれる）|
| `PendingUpdateOperations` | `TSet<FPrimitiveSceneInfo*>` | 更新待ちプリミティブ |
| `PendingRemoveOperations` | `TArray<FLumenPrimitiveGroupRemoveInfo>` | 削除待ちプリミティブ（遅延削除）|
| `bReuploadSceneRequest` | `bool` | 全データの GPU 再アップロードを要求するフラグ |
| `CardSharingInfoMap` | `TMap<FLumenCardId, TSparseArray<FLumenCardSharingInfo>>` | Card 共有情報のマップ（同一 ID → 共有 Card リスト）|

---

## FLumenSceneData::AddMeshCards

```cpp
void AddMeshCards(int32 PrimitiveGroupIndex);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `PrimitiveGroupIndex` | `int32` | Card を追加する `FLumenPrimitiveGroup` のインデックス |

### 使用箇所
- [[ref_lumen_scene]] `UpdateLumenScenePrimitives()` — プリミティブ追加処理の末尾で呼ばれる

### 内部処理フロー

1. **既割り当てチェック**
   ```cpp
   if (PrimitiveGroup.MeshCardsIndex != -1) return; // すでに Card あり
   ```

2. **Card ビルドデータの取得**
   ```cpp
   if (PrimitiveGroup.bHeightfield) {
       BuildMeshCardsDataForHeightfield(...);
   } else if (PrimitiveGroup.HasMergedInstances()) {
       BuildMeshCardsDataForMergedInstances(...);
   } else {
       // PrimitiveSceneProxy から FMeshCardsBuildData を取得
   }
   ```

3. **Card スパンの確保と初期化**
   ```cpp
   AddMeshCardsFromBuildData(PrimitiveGroupIndex, LocalToWorld, MeshCardsBuildData, PrimitiveGroup);
   ```

4. **カリング情報の更新**
   ```cpp
   if (PrimitiveGroup.PrimitiveCullingInfoIndex != INDEX_NONE) {
       CullingInfo.bVisible = PrimitiveGroup.bValidMeshCards;
   }
   ```

---

## FLumenSceneData::RemoveMeshCards

```cpp
void RemoveMeshCards(int32 PrimitiveGroupIndex, bool bUpdateCullingInfo = true);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `PrimitiveGroupIndex` | `int32` | Card を削除する PrimitiveGroup のインデックス |
| `bUpdateCullingInfo` | `bool` | true のとき GPU Driven カリング情報も更新する（デフォルト true）|

### 使用箇所
- [[ref_lumen_scene]] `UpdateLumenScenePrimitives()` — プリミティブ削除処理で呼ばれる
- [[ref_lumen_scene_data]] `FLumenSceneData::AddMeshCards()` — 再ビルド前に旧データを削除する際にも呼ばれる

### 内部処理フロー

1. **全ページの解放**
   ```cpp
   for (int32 CardIndex = FirstCardIndex; CardIndex < FirstCardIndex + NumCards; CardIndex++) {
       RemoveCardFromAtlas(CardIndex); // 物理ページを SurfaceCacheAllocator.Free() で解放
   }
   ```

2. **ハイトフィールドの削除**
   ```cpp
   if (PrimitiveGroup.HeightfieldIndex != -1) {
       Heightfields.RemoveSpan(PrimitiveGroup.HeightfieldIndex, 1);
   }
   ```

3. **GPU バッファ更新のスケジュール**
   ```cpp
   MeshCardsIndicesToUpdateInBuffer.Add(MeshCardsIndex);
   ```

4. **スパンの解放**
   ```cpp
   Cards.RemoveSpan(MeshCardsInstance.FirstCardIndex, MeshCardsInstance.NumCards);
   MeshCards.RemoveSpan(PrimitiveGroup.MeshCardsIndex, 1);
   PrimitiveGroup.MeshCardsIndex = -1;
   ```

---

> [!note]- FLumenSceneData::UpdateMeshCards — トランスフォーム更新
> 
> ```cpp
> void UpdateMeshCards(
>     const FMatrix& LocalToWorld,
>     int32 MeshCardsIndex,
>     const FMeshCardsBuildData& MeshCardsBuildData);
> ```
> 
> **パラメータ**
> 
> | 引数 | 型 | 説明 |
> |------|-----|------|
> | `LocalToWorld` | `const FMatrix&` | 新しいローカル→ワールド変換行列 |
> | `MeshCardsIndex` | `int32` | 更新対象 MeshCards のインデックス |
> | `MeshCardsBuildData` | `const FMeshCardsBuildData&` | 新しいビルドデータ（境界更新に使用）|
> 
> **使用箇所**: [[ref_lumen_scene]] `UpdateLumenScenePrimitives()` — `PendingUpdateOperations` の処理で呼ばれる
> 
> **内部処理フロー**
> 1. `MeshCardsInstance.SetTransform(LocalToWorld)` で MeshCards の行列を更新
> 2. 全 Card に対して `Card.SetTransform(LocalToWorld, MeshCardsInstance)` を呼び WorldOBB を更新
> 3. `CardIndicesToUpdateInBuffer.Add(CardIndex)` で GPU 同期をスケジュール

> [!note]- FLumenSceneData::ReallocVirtualSurface — 仮想サーフェス確保
> 
> ```cpp
> void ReallocVirtualSurface(FLumenCard& Card, int32 CardIndex, int32 ResLevel, bool bLockPages);
> ```
> 
> **パラメータ**
> 
> | 引数 | 型 | 説明 |
> |------|-----|------|
> | `Card` | `FLumenCard&` | 確保対象の Card |
> | `CardIndex` | `int32` | Card のシーンインデックス |
> | `ResLevel` | `int32` | 確保する解像度レベル |
> | `bLockPages` | `bool` | true のとき確保したページを LRU から除外してロック |
> 
> **使用箇所**: [[ref_lumen_surface_cache]] `UpdateSurfaceCacheAllocationState()` 内で呼ばれる

> [!note]- FLumenSceneData::FreeVirtualSurface — 仮想サーフェス解放
> 
> ```cpp
> void FreeVirtualSurface(FLumenCard& Card, uint8 FromResLevel, uint8 ToResLevel);
> ```
> 
> **パラメータ**
> 
> | 引数 | 型 | 説明 |
> |------|-----|------|
> | `Card` | `FLumenCard&` | 解放対象の Card |
> | `FromResLevel` | `uint8` | 解放開始解像度レベル（含む）|
> | `ToResLevel` | `uint8` | 解放終了解像度レベル（含まない）|
> 
> **使用箇所**: [[ref_lumen_surface_cache]] `FreeSurfaceCacheAllocation()` から呼ばれる

> [!note]- FLumenSceneData::EvictOldestAllocation — LRU eviction
> 
> ```cpp
> bool EvictOldestAllocation(
>     uint32 MaxFramesSinceLastUsed,
>     TSparseUniqueList<int32, SceneRenderingAllocator>& DirtyCards);
> ```
> 
> **パラメータ**
> 
> | 引数 | 型 | 説明 |
> |------|-----|------|
> | `MaxFramesSinceLastUsed` | `uint32` | この値より古いページを evict 対象とする |
> | `DirtyCards` | `TSparseUniqueList<int32>&` | evict によって dirty になった Card インデックスを積む |
> 
> **戻り値**: `bool` — evict できたページがあれば true
> 
> **使用箇所**: [[ref_lumen_surface_cache]] `UpdateSurfaceCacheAllocationState()` — アトラスが満杯のとき古いページを解放して空きを作る

> [!note]- FLumenSceneData::AllocateCardAtlases — アトラス生成
> 
> ```cpp
> void AllocateCardAtlases(
>     FRDGBuilder& GraphBuilder,
>     FLumenSceneFrameTemporaries& FrameTemporaries,
>     const FSceneViewFamily* ViewFamily);
> ```
> 
> **パラメータ**
> 
> | 引数 | 型 | 説明 |
> |------|-----|------|
> | `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー（テクスチャ登録用）|
> | `FrameTemporaries` | `FLumenSceneFrameTemporaries&` | 出力: 各アトラスの RDG テクスチャ参照を書き込む |
> | `ViewFamily` | `const FSceneViewFamily*` | 圧縮設定・プラットフォーム情報の参照用 |
> 
> **使用箇所**: [[ref_lumen_scene]] `UpdateLumenScene()` の冒頭で毎フレーム呼ばれる（`bReallocateAtlas` が true のときのみ実際に再生成）
> 
> **内部処理フロー**
> 1. 物理アトラスサイズを `GetPhysicalAtlasSize()` で取得（`r.LumenScene.SurfaceCache.AtlasSize` 由来）
> 2. `ESurfaceCacheCompression` に基づきアルベド・法線・エミッシブのフォーマットを選択
> 3. 各アトラステクスチャを `RegisterExternalTexture()` または `CreateTexture()` で RDG に登録
> 4. `FrameTemporaries.AlbedoAtlas = AlbedoAtlas;` ... と各フィールドに書き込む

---

## ESurfaceCacheCompression

```cpp
enum class ESurfaceCacheCompression : uint8 {
    Disabled,             // 圧縮なし（最も互換性が高い）
    UAVAliasing,          // UAV エイリアシングで直接圧縮フォーマットに書き込む（DX12 のみ）
    CopyTextureRegion,    // 非圧縮で描画後に CopyTextureRegion で圧縮（一部プラットフォーム）
    FramebufferCompression // フレームバッファ圧縮（モバイル向け）
};

ESurfaceCacheCompression GetSurfaceCacheCompression();
```

### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::AllocateCardAtlases()` — アトラスフォーマット選択
- [[ref_lumen_scene]] `UpdateLumenSurfaceCacheAtlas()` — キャプチャデータのコピー方法選択
