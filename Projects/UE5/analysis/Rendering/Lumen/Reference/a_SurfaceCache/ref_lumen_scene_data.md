# リファレンス：LumenSceneData.h

- グループ: a - Surface Cache
- 上位: [[a_lumen_surface_cache]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneData.h`

---

## 概要

Lumen の Surface Cache を構成するすべての**コアデータ構造**を定義するヘッダ。  
Card・PageTable・アロケータ・シーン全体の状態を管理するクラス群が含まれる。

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

    bool IsValid() const;
    void Invalidate();
    static constexpr FLumenCardId GetInvalidId();
};
```

---

## FLumenCard

Surface Cache の基本単位。メッシュ表面の 1 方向の平面パッチ。

```cpp
class FLumenCard {
    FLumenCardOBBf LocalOBB;      // ローカル空間 OBB
    FLumenCardOBBd WorldOBB;      // ワールド空間 OBB（double 精度）
    FLumenCardOBBf MeshCardsOBB;  // MeshCards 空間の OBB

    bool bVisible;
    bool bHeightfield;
    bool bAxisXFlipped;
    ELumenCardDilationMode DilationMode;

    uint8 MinAllocatedResLevel;   // 割り当て済みの最小解像度レベル
    uint8 MaxAllocatedResLevel;   // 割り当て済みの最大解像度レベル
    uint8 DesiredLockedResLevel;  // 距離から算出した希望解像度レベル

    // 9 段分のミップマップ割り当て [ResLevel - MinResLevel]
    FLumenSurfaceMipMap SurfaceMipMaps[Lumen::NumResLevels];

    int32 MeshCardsIndex;
    int32 IndexInMeshCards;
    uint8 AxisAlignedDirectionIndex; // 0〜5（±X/Y/Z）
    float ResolutionScale;
    float CardAspect;              // アスペクト比（初期値から変更不可）

    // Card 共有（同一 Card を複数 MeshCards で共有する仕組み）
    FLumenCardId CardSharingId;
    int32 CardSharingListIndex;
};
```

**主要メソッド:**

| メソッド | 説明 |
|---------|------|
| `Initialize(...)` | Card を初期化 |
| `SetTransform(LocalToWorld, MeshCards)` | トランスフォームを更新 |
| `UpdateMinMaxAllocatedLevel()` | 割り当て済み解像度範囲を再計算 |
| `IsAllocated()` | 物理ページが割り当て済みか |
| `GetMipMap(ResLevel)` | 指定解像度レベルの MipMap を取得 |
| `GetMipMapDesc(ResLevel, Desc)` | MipMap の詳細記述を取得 |
| `GetSurfaceStats(PageTable, Stats)` | テクセル使用統計を取得 |
| `ResLevelToResLevelXYBias()` | 解像度レベルの XY バイアスを返す |

---

## FLumenPrimitiveGroup

1 プリミティブ（またはインスタンスグループ）に対応する Lumen 管理単位。

```cpp
class FLumenPrimitiveGroup {
    TArray<FPrimitiveSceneInfo*> Primitives;  // 対象プリミティブ
    int32 MeshCardsIndex;      // FLumenMeshCards へのインデックス
    int32 HeightfieldIndex;    // ハイトフィールドインデックス
    uint32 CustomId;           // 一意ID
    float CardResolutionScale; // 解像度スケール

    uint8 bValidMeshCards : 1; // MeshCards が有効か
    uint8 bFarField : 1;       // 遠距離フィールド扱いか
    uint8 bHeightfield : 1;    // ハイトフィールドか
    uint8 bEmissiveLightSource : 1; // 自発光光源か
    uint8 bOpaqueOrMasked : 1; // 不透明またはマスクか
    uint8 bLandscape : 1;      // ランドスケープか
    uint8 LightingChannelMask; // ライティングチャンネルマスク

    bool HasMergedInstances() const;
    bool HasMergedPrimitives() const;
};
```

---

## FLumenPageTableEntry

物理ページとアトラス矩形の対応情報。

```cpp
struct FLumenPageTableEntry {
    FIntPoint PhysicalPageCoord;   // 物理ページ座標（-1,-1 = 未割り当て）
    FIntRect  PhysicalAtlasRect;   // アトラス上の矩形

    // サンプリング用（粗いページへの参照が可能）
    uint32 SamplePageIndex;
    uint16 SampleAtlasBiasX, SampleAtlasBiasY;
    uint16 SampleCardResLevelX, SampleCardResLevelY;

    int32 CardIndex;
    uint8 ResLevel;
    FVector4f CardUVRect;
    FIntPoint SubAllocationSize;   // サブアロケーションサイズ（-1 = 非サブアロケーション）

    bool IsSubAllocation() const;
    bool IsMapped() const;
    uint32 GetNumVirtualTexels() const;
    uint32 GetNumPhysicalTexels() const;
};
```

---

## FLumenSurfaceCacheAllocator

物理ページのビン方式アロケータ。小サイズ要素はサブアロケーションで詰め込む。

```cpp
class FLumenSurfaceCacheAllocator {
    struct FAllocation {
        FIntPoint PhysicalPageCoord;
        FIntRect  PhysicalAtlasRect;
    };

    void Init(const FIntPoint& InPageAtlasSizeInPages);
    void Allocate(const FLumenPageTableEntry& Page, FAllocation& Allocation);
    void Free(const FLumenPageTableEntry& Page);
    bool IsSpaceAvailable(const FLumenCard& Card, int32 ResLevel, bool bSinglePage) const;
    void GetStats(FStats& Stats) const;
};
```

**ビン構造:**
- 物理ページ 128×128 をサブアロケーションサイズ別の **PageBin** で管理
- PageBin のサイズは 8×8、8×16、8×32、...、128×128 の最大 64 種
- `FPageBinLookup` で要素サイズ → PageBin を O(1) で引く

---

## FLumenSceneFrameTemporaries

1 フレーム中のみ有効な RDG テクスチャ・バッファ群。全アトラスへの参照を保持。

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

    // GPU バッファの SRV/UAV
    FRDGBufferSRV* CardBufferSRV;
    FRDGBufferSRV* MeshCardsBufferSRV;
    FRDGBufferSRV* PageTableBufferSRV;
    FRDGBufferSRV* CardPageBufferSRV;
    FRDGBufferUAV* CardPageBufferUAV;
    FRDGBufferUAV* CardPageLastUsedBufferUAV;

    // Surface Cache フィードバックリソース
    FLumenSurfaceCacheFeedback::FFeedbackResources SurfaceCacheFeedbackResources;
};
```

---

## FLumenViewOrigin

Lumen シーン更新用のビュー原点情報。キューブマップなど共有ビューをサポート。

```cpp
struct FLumenViewOrigin {
    FVector LumenSceneViewOrigin;          // Lumen シーンの視点原点
    FVector4f WorldCameraOrigin;           // ワールドカメラ原点
    FMatrix44f FrustumTranslatedWorldToClip; // 視錐台変換行列
    float OrthoMaxDimension;               // 正投影の最大次元（0 = 透視投影）
    float MaxTraceDistance;                // 最大トレース距離
    float CardMaxDistance;                 // Card の最大距離
    const FViewInfo* ReferenceView;        // 参照用ビュー（後方互換のため保持）

    void Init(const FViewInfo& View);
    bool IsPerspectiveProjection() const;
};
```

---

## ESurfaceCacheCompression

```cpp
enum class ESurfaceCacheCompression : uint8 {
    Disabled,
    UAVAliasing,
    CopyTextureRegion,
    FramebufferCompression
};

ESurfaceCacheCompression GetSurfaceCacheCompression(); // 現在の圧縮方式を返す
```

---

## FLumenSharedRT

RDG テクスチャを再利用するためのラッパー。解像度変化に追従してリサイズする。

```cpp
class FLumenSharedRT {
    FRDGTextureRef CreateSharedRT(FRDGBuilder&, const FRDGTextureDesc&, FIntPoint VisibleExtent, const TCHAR* Name, ERDGTextureFlags);
    FRDGTextureRef GetRenderTarget() const;
};
```
