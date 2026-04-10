# リファレンス：LumenMeshCards.h / LumenMeshCards.cpp

- グループ: a - Surface Cache
- 上位: [[a_lumen_surface_cache]]
- 関連: [[ref_lumen_scene_data]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenMeshCards.h/cpp`

---

## 概要

1 メッシュに紐づく **Card（平面パッチ）群** を管理するクラス。  
メッシュの OBB を 6 方向（±X/Y/Z）に投影して Card を生成・更新する処理を担う。

---

## FLumenMeshCards

> **概要**: 1 プリミティブグループに対応する MeshCards インスタンス。Card スパンへのインデックスと 6 方向ルックアップテーブルを保持する。

```cpp
class FLumenMeshCards {
public:
    FMatrix LocalToWorld;
    FVector3f LocalToWorldScale;
    FMatrix WorldToLocalRotation;
    FBox LocalBounds;

    int32 PrimitiveGroupIndex;
    bool bFarField;
    bool bHeightfield;
    bool bMostlyTwoSided;
    bool bEmissiveLightSource;

    uint32 FirstCardIndex;
    uint32 NumCards;
    uint32 CardLookup[Lumen::NumAxisAlignedDirections]; // NumAxisAlignedDirections = 6

    TArray<int32, TInlineAllocator<1>> ScenePrimitiveIndices;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `LocalToWorld` | `FMatrix` | ローカル→ワールド変換行列（double 精度列なし float 行列）|
| `LocalToWorldScale` | `FVector3f` | スケール成分の抽出値（Card OBB サイズ計算に使用）|
| `WorldToLocalRotation` | `FMatrix` | ワールド→ローカル回転行列（MeshCardsOBB 計算用）|
| `LocalBounds` | `FBox` | ローカル空間 AABB（世界範囲計算用）|
| `PrimitiveGroupIndex` | `int32` | 親 `FLumenPrimitiveGroup` のインデックス |
| `bFarField` | `bool` | 遠距離フィールド扱いか（`r.LumenScene.FarField` 由来）|
| `bHeightfield` | `bool` | ランドスケープハイトフィールドか |
| `bMostlyTwoSided` | `bool` | マテリアルの多数が両面か（Card の法線向き判定に影響）|
| `bEmissiveLightSource` | `bool` | 自発光光源として GI に寄与するか（Card の最小サイズ閾値が変わる）|
| `FirstCardIndex` | `uint32` | `FLumenSceneData::Cards` 内での先頭インデックス |
| `NumCards` | `uint32` | 所有する Card の枚数（最大 6）|
| `CardLookup[6]` | `uint32[6]` | 方向インデックス（0〜5）→ Card ビットマスクのルックアップ |
| `ScenePrimitiveIndices` | `TArray<int32, TInlineAllocator<1>>` | GPU Scene 上のプリミティブインデックス（インスタンス時は複数）|

### CardLookup の方向インデックス

```
0 = +X    1 = -X
2 = +Y    3 = -Y
4 = +Z    5 = -Z

CardLookup[方向] = そちらを向いた Card の（MeshCards 内）ビットマスク
例: CardLookup[0] = 0b000001 → 0番目のCardが+X方向
    CardLookup[4] = 0b000100 → 2番目のCardが+Z方向
```

---

## FLumenMeshCards::Initialize

```cpp
void Initialize(
    const FMatrix& InLocalToWorld,
    int32 InPrimitiveGroupIndex,
    uint32 InFirstCardIndex,
    uint32 InNumCards,
    const FMeshCardsBuildData& MeshCardsBuildData,
    const FLumenPrimitiveGroup& PrimitiveGroup);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `InLocalToWorld` | `const FMatrix&` | 初期ローカル→ワールド変換行列 |
| `InPrimitiveGroupIndex` | `int32` | 親 PrimitiveGroup のシーンインデックス |
| `InFirstCardIndex` | `uint32` | `FLumenSceneData::Cards` でのスパン先頭インデックス |
| `InNumCards` | `uint32` | Card の枚数 |
| `MeshCardsBuildData` | `const FMeshCardsBuildData&` | ビルド時に計算された OBB・Card リスト |
| `PrimitiveGroup` | `const FLumenPrimitiveGroup&` | フラグ（bFarField, bHeightfield 等）のコピー元 |

### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::AddMeshCardsFromBuildData()` — MeshCards スパン確保後に呼ばれる

### 内部処理フロー

1. **PrimitiveGroupIndex と Card スパン情報の保存**
   ```cpp
   PrimitiveGroupIndex = InPrimitiveGroupIndex;
   FirstCardIndex = InFirstCardIndex;
   NumCards       = InNumCards;
   ```

2. **フラグのコピー**
   ```cpp
   bFarField          = PrimitiveGroup.bFarField;
   bHeightfield       = PrimitiveGroup.bHeightfield;
   bEmissiveLightSource = PrimitiveGroup.bEmissiveLightSource;
   bMostlyTwoSided    = MeshCardsBuildData.bMostlyTwoSided;
   LocalBounds        = MeshCardsBuildData.Bounds;
   ```

3. **トランスフォームの初期設定**
   ```cpp
   SetTransform(InLocalToWorld);
   ```

4. **CardLookup の初期化（全方向を無効に）**
   ```cpp
   FMemory::Memset(CardLookup, 0, sizeof(CardLookup));
   // 実際の値は UpdateLookup() で後から設定される
   ```

---

## FLumenMeshCards::UpdateLookup

```cpp
void UpdateLookup(const TSparseSpanArray<FLumenCard>& Cards);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `Cards` | `const TSparseSpanArray<FLumenCard>&` | シーン全体の Card 疎配列 |

### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::AddMeshCardsFromBuildData()` — Card 初期化後に呼ばれる
- [[ref_lumen_scene_data]] `FLumenSceneData::UpdateMeshCards()` — Card が追加・削除されたときに再構築

### 内部処理フロー

1. **全方向のビットマスクをリセット**
   ```cpp
   FMemory::Memset(CardLookup, 0, sizeof(CardLookup));
   ```

2. **各 Card のビットマスクを設定**
   ```cpp
   for (uint32 LocalCardIndex = 0; LocalCardIndex < NumCards; LocalCardIndex++) {
       const FLumenCard& Card = Cards[FirstCardIndex + LocalCardIndex];
       // その方向の MeshCards 内ビット位置を立てる
       CardLookup[Card.AxisAlignedDirectionIndex] |= (1u << LocalCardIndex);
   }
   ```

---

## FLumenMeshCards::SetTransform

```cpp
void SetTransform(const FMatrix& InLocalToWorld);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `InLocalToWorld` | `const FMatrix&` | 新しいローカル→ワールド変換行列 |

### 使用箇所
- [[ref_lumen_scene_data]] `FLumenMeshCards::Initialize()` — 初期化時
- [[ref_lumen_scene_data]] `FLumenSceneData::UpdateMeshCards()` — トランスフォーム更新時

### 内部処理フロー

```cpp
LocalToWorld = InLocalToWorld;
LocalToWorldScale = FVector3f(
    InLocalToWorld.GetScaledAxis(EAxis::X).Size(),
    InLocalToWorld.GetScaledAxis(EAxis::Y).Size(),
    InLocalToWorld.GetScaledAxis(EAxis::Z).Size());
// 回転のみの逆行列（スケールを除去してから転置）
WorldToLocalRotation = FMatrix(
    InLocalToWorld.GetUnitAxis(EAxis::X),
    InLocalToWorld.GetUnitAxis(EAxis::Y),
    InLocalToWorld.GetUnitAxis(EAxis::Z),
    FVector::ZeroVector).GetTransposed();
```

---

<details>
<summary>FLumenMeshCards::GetWorldSpaceBounds — ワールド空間 AABB 取得</summary>

```cpp
FBox GetWorldSpaceBounds() const;
```

### 戻り値
`FBox` — `LocalBounds` を `LocalToWorld` で変換したワールド空間 AABB

### 使用箇所
- [[ref_lumen_scene_gpu_driven_update]] `GPUDrivenUpdate()` — プリミティブのワールド AABB をフラスタム判定に使用

</details>

---

## namespace LumenMeshCards

### GetCardMinSurfaceArea

```cpp
namespace LumenMeshCards {
    float GetCardMinSurfaceArea(bool bEmissiveLightSource);
}
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `bEmissiveLightSource` | `bool` | true = 自発光光源用の小さい閾値を返す |

### 戻り値
`float` — Card の最小表面積（cm²）。この値未満の Card は生成されない。

### 内部動作
```cpp
// bEmissiveLightSource == false（通常）
return CVarLumenSceneCardMinSurfaceArea.GetValueOnRenderThread();

// bEmissiveLightSource == true（自発光は小さい Card も有効）
return CVarLumenSceneEmissiveLightSourceCardMinSurfaceArea.GetValueOnRenderThread();
```

### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::AddMeshCardsFromBuildData()` — Card フィルタリング条件として使用

---

## namespace Lumen — UpdateCardSceneBuffer

```cpp
namespace Lumen {
    void UpdateCardSceneBuffer(
        FRDGBuilder& GraphBuilder,
        FRDGScatterUploadBuilder& UploadBuilder,
        FLumenSceneFrameTemporaries& FrameTemporaries,
        const FSceneViewFamily& ViewFamily,
        FScene* Scene);
}
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー（バッファ登録・パス追加用）|
| `UploadBuilder` | `FRDGScatterUploadBuilder&` | Scatter Upload パターンでの GPU バッファ更新用ビルダー |
| `FrameTemporaries` | `FLumenSceneFrameTemporaries&` | 出力: 更新後の SRV/UAV を書き込む |
| `ViewFamily` | `const FSceneViewFamily&` | フォーマット・プラットフォーム判定用 |
| `Scene` | `FScene*` | `FLumenSceneData` を含むシーン |

### 使用箇所
- [[ref_lumen_scene]] `UpdateLumenScene()` / `FDeferredShadingRenderer::UpdateLumenScene()` — Lumen シーン更新の最終ステップで呼ばれる

### 内部処理フロー

1. **CardBuffer の更新**（`CardIndicesToUpdateInBuffer` に積まれた Card のみスキャッタアップロード）
   ```cpp
   for (int32 CardIndex : LumenData.CardIndicesToUpdateInBuffer) {
       FLumenCardGPUData::FillData(LumenData.Cards[CardIndex], ...);
       UploadBuilder.Add(CardIndex, GPUData); // 変更分だけ書き込む
   }
   LumenData.CardIndicesToUpdateInBuffer.Reset();
   ```

2. **MeshCardsBuffer の更新**（同様のパターン）
   ```cpp
   for (int32 MeshCardsIndex : LumenData.MeshCardsIndicesToUpdateInBuffer) {
       FLumenMeshCardsGPUData::FillData(...);
       UploadBuilder.Add(MeshCardsIndex, GPUData);
   }
   ```

3. **PageTableBuffer の更新**
   ```cpp
   // PageTable の変更エントリをアップロード
   ```

4. **Uniform Buffer の作成**
   ```cpp
   // FLumenCardScene Uniform Buffer を生成して FrameTemporaries に書き込む
   FrameTemporaries.LumenCardSceneUniformBuffer = CreateUniformBufferImmediate(CardSceneParameters, ...);
   ```

5. **SRV / UAV の登録**
   ```cpp
   FrameTemporaries.CardBufferSRV     = GraphBuilder.CreateSRV(CardBuffer);
   FrameTemporaries.MeshCardsBufferSRV = GraphBuilder.CreateSRV(MeshCardsBuffer);
   // ... 他の SRV/UAV
   ```

---

## FMeshCardsBuildData（参考）

ビルド時（CPU）に生成される Card の定義データ。`FLumenMeshCards::Initialize()` に渡される。

```cpp
struct FMeshCardsBuildData {
    FBox Bounds;                          // ローカル空間 AABB
    bool bMostlyTwoSided;                 // 両面マテリアルが多いか
    TArray<FLumenCardBuildData> Cards;    // 各 Card の OBB・方向情報
};

struct FLumenCardBuildData {
    FLumenCardOBBf OBB;                   // Card のローカル空間 OBB
    uint8 AxisAlignedDirectionIndex;      // 方向インデックス（0〜5）
};
```

### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::AddMeshCardsFromBuildData()` — カードビルドデータから Card を生成する際に参照
- プロキシが `GetMeshCardsBuildData()` を実装してビルド時に生成する

---

## 定数・CVar

```cpp
namespace Lumen {
    constexpr uint32 NumAxisAlignedDirections = 6; // ±X/Y/Z の 6 方向
}
```

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.LumenScene.SurfaceCache.AtlasSize` | 4096 | Surface Cache アトラスの一辺サイズ（px）|
| `r.LumenScene.FarField` | 0 | 遠距離フィールド有効/無効 |
| `r.LumenScene.FarField.MaxTraceDistance` | 1.0e6 | 遠距離フィールドの最大トレース距離（cm）|
| `r.LumenScene.FarField.FarFieldDitherScale` | 200.0 | 近距離/遠距離の境界ディザ幅（cm）|
| `r.LumenScene.GlobalSDF.Resolution` | 252 | Global SDF の解像度 |
| `r.LumenScene.GlobalSDF.ClipmapExtent` | 2500.0 | 最初のクリップマップの半径（cm）|
