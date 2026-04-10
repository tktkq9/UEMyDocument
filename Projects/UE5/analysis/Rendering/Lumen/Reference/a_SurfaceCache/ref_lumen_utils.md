# リファレンス：LumenSparseSpanArray.h / LumenUniqueList.h

- グループ: a - Surface Cache
- 上位: [[a_lumen_surface_cache]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSparseSpanArray.h`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenUniqueList.h`

---

## 概要

Lumen 内部で使われる**汎用コンテナユーティリティ**。  
`TSparseSpanArray` は Card や MeshCards の疎な配列管理に、  
`TSparseUniqueList` / `FUniqueIndexList` は重複排除リストの管理に使われる。

---

## TSparseSpanArray\<ElementType\>

> **概要**: **疎な配列 + 安定インデックス + 連続スパン割り当て**のコンテナ。Card や MeshCards など「インデックスが外部から参照され、途中削除が起きる」データの管理に最適。

```cpp
template <typename ElementType>
class TSparseSpanArray {
public:
    int32 Num() const;
    void Reserve(int32 NumElements);

    int32 AddSpan(int32 NumElements);
    void RemoveSpan(int32 FirstElementIndex, int32 NumElements);

    bool IsAllocated(int32 Index) const;
    ElementType& operator[](int32 Index);
    const ElementType& operator[](int32 Index) const;

    // イテレーション
    auto begin();
    auto end();
};
```

### メンバ変数（内部）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Data` | `TArray<ElementType>` | 要素データの実体配列（穴も含む連続バッファ）|
| `AllocationMask` | `TBitArray<>` | インデックスが割り当て済みかのビット配列 |
| `FreeSpans` | `TArray<FSpan>` | 解放済みスパンのリスト（再利用候補）|

### AddSpan

```cpp
int32 AddSpan(int32 NumElements);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `NumElements` | `int32` | 確保したい連続スロット数 |

#### 戻り値
`int32` — 確保した連続スロットの先頭インデックス

#### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::AddMeshCardsFromBuildData()` — Card スパンと MeshCards スパンの確保
- [[ref_lumen_scene_data]] `FLumenSceneData::ReallocVirtualSurface()` — PageTable スパンの確保

#### 内部動作
```cpp
int32 TSparseSpanArray::AddSpan(int32 NumElements)
{
    // 解放済みスパンから NumElements 以上の空きを探す
    for (FSpan& FreeSpan : FreeSpans) {
        if (FreeSpan.Num >= NumElements) {
            int32 FirstIndex = FreeSpan.FirstIndex;
            FreeSpan.FirstIndex += NumElements;
            FreeSpan.Num       -= NumElements;
            AllocationMask.SetRange(FirstIndex, NumElements, true);
            return FirstIndex;
        }
    }
    // 見つからなければ末尾に拡張
    int32 FirstIndex = Data.AddDefaulted(NumElements);
    AllocationMask.Add(true, NumElements);
    return FirstIndex;
}
```

---

### RemoveSpan

```cpp
void RemoveSpan(int32 FirstElementIndex, int32 NumElements);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `FirstElementIndex` | `int32` | 解放するスパンの先頭インデックス |
| `NumElements` | `int32` | 解放するスロット数 |

#### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::RemoveMeshCards()` — Card スパン・MeshCards スパンの解放
- [[ref_lumen_surface_cache]] `FreeSurfaceCacheAllocation()` — PageTable スパンの解放

#### 内部動作
```cpp
void TSparseSpanArray::RemoveSpan(int32 FirstElementIndex, int32 NumElements)
{
    // デストラクタを呼んでデフォルト状態に戻す
    for (int32 i = 0; i < NumElements; i++) {
        Data[FirstElementIndex + i] = ElementType{};
    }
    AllocationMask.SetRange(FirstElementIndex, NumElements, false);
    // FreeSpans に追加（隣接スパンとマージも試みる）
    FreeSpans.Add({FirstElementIndex, NumElements});
}
```

---

> [!note]- IsAllocated — 指定インデックスが割り当て済みか
> 
> ```cpp
> bool IsAllocated(int32 Index) const;
> ```
> 
> **内部動作**: `return AllocationMask[Index];`
> 
> **使用箇所**: デバッグ検証（`check(IsAllocated(CardIndex))`）やイテレーション時の穴スキップ

### 使用例

```
FLumenMeshCards が 3 枚の Card を持つ場合:
  AddSpan(3) → 先頭インデックス 10, 11, 12 を確保
  MeshCards.FirstCardIndex = 10
  MeshCards.NumCards = 3

別の MeshCards が削除されても、インデックス 10〜12 は変わらない（安定インデックス）
RemoveSpan(5, 2) → インデックス 5, 6 が空きスパンに戻る
```

### 使用箇所まとめ

| インスタンス | 型 | 説明 |
|------------|-----|------|
| `FLumenSceneData::Cards` | `TSparseSpanArray<FLumenCard>` | シーン全 Card |
| `FLumenSceneData::MeshCards` | `TSparseSpanArray<FLumenMeshCards>` | シーン全 MeshCards |
| `FLumenSceneData::PrimitiveGroups` | `TSparseSpanArray<FLumenPrimitiveGroup>` | シーン全 PrimitiveGroup |
| `FLumenSceneData::Heightfields` | `TSparseSpanArray<FLumenHeightfield>` | ハイトフィールド |
| `FLumenSceneData::PageTable` | `TSparseSpanArray<FLumenPageTableEntry>` | 仮想ページテーブル |

---

## FUniqueIndexList

> **概要**: **フラット配列 + ビットセット**による重複排除インデックスリスト。整数インデックスに特化した軽量版。ビットセットで重複チェックを O(1) で行う。

```cpp
struct FUniqueIndexList {
public:
    void Add(int32 Index);

    int32 Num() const;
    void Reset();
    void Empty(int32 Slack = 0);

    // 範囲 for でイテレーション可能
    auto begin() const;
    auto end() const;

private:
    TArray<int32>    Indices;
    TBitArray<>      IndicesMarkedToUpdate;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Indices` | `TArray<int32>` | 重複なしのインデックスリスト（イテレーション用）|
| `IndicesMarkedToUpdate` | `TBitArray<>` | インデックスが追加済みかのビット配列（重複防止用）|

### Add

```cpp
void Add(int32 Index);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `Index` | `int32` | 追加するインデックス（重複は無視される）|

#### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::CardIndicesToUpdateInBuffer` への追加 — Card の GPU バッファ更新をスケジュール
- [[ref_lumen_scene_data]] `FLumenSceneData::MeshCardsIndicesToUpdateInBuffer` への追加

#### 内部動作
```cpp
void FUniqueIndexList::Add(int32 Index)
{
    // ビットセットを必要に応じて拡張
    if (Index >= IndicesMarkedToUpdate.Num()) {
        IndicesMarkedToUpdate.Add(false, Index - IndicesMarkedToUpdate.Num() + 1);
    }
    // 未追加なら追加
    if (!IndicesMarkedToUpdate[Index]) {
        IndicesMarkedToUpdate[Index] = true;
        Indices.Add(Index);
    }
    // すでに追加済みならスキップ（重複排除）
}
```

---

> [!note]- Reset / Empty — リストのリセット
> 
> ```cpp
> void Reset();         // 全リセット（メモリ解放なし）
> void Empty(int32 Slack = 0); // クリア（Slack 分の容量を確保）
> ```
> 
> **内部動作**
> 
> ```cpp
> void Reset()
> {
>     for (int32 Index : Indices) {
>         IndicesMarkedToUpdate[Index] = false;
>     }
>     Indices.Reset();
> }
> ```
> 
> **使用箇所**: [[ref_lumen_mesh_cards]] `Lumen::UpdateCardSceneBuffer()` — GPU アップロード完了後にリセット

### 使用例

```cpp
FUniqueIndexList DirtyCardIndices;

// 同じ Card が複数回 dirty になっても 1 回だけ更新する
DirtyCardIndices.Add(CardIndex); // 1回目: 追加
DirtyCardIndices.Add(CardIndex); // 2回目: 重複排除でスキップ

// 全 dirty Card を処理
for (int32 CardIndex : DirtyCardIndices) {
    UpdateCard(CardIndex);
}
DirtyCardIndices.Reset();
```

---

## TSparseUniqueList\<ElementType, Allocator\>

> **概要**: **フラット配列 + ハッシュセット**による重複排除リスト。重複チェックを O(1) で行いつつ、イテレーションはフラット配列で高速に行える。`FUniqueIndexList` より汎用的だが若干重い。

```cpp
template <typename ElementType, typename Allocator>
struct TSparseUniqueList {
    void Add(ElementType Element);

    TArray<ElementType, Allocator> Array;
    Experimental::TSherwoodSet<ElementType> Set;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Array` | `TArray<ElementType, Allocator>` | 追加順のフラット配列（イテレーション用）|
| `Set` | `TSherwoodSet<ElementType>` | 重複チェック用ハッシュセット（O(1) 探索）|

### Add

```cpp
void Add(ElementType Element);
```

#### 内部動作
```cpp
if (Set.Find(Element) == nullptr) {
    Set.Add(Element);
    Array.Add(Element);
}
```

#### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::EvictOldestAllocation()` — evict によって dirty になった Card インデックスを積む
- [[ref_lumen_scene_gpu_driven_update]] GPU Driven な追加/削除リストの管理

---

## 使用箇所まとめ

| コンテナ | 主な使用箇所 |
|---------|------------|
| `TSparseSpanArray<FLumenCard>` | [[ref_lumen_scene_data]] `FLumenSceneData::Cards` |
| `TSparseSpanArray<FLumenMeshCards>` | [[ref_lumen_scene_data]] `FLumenSceneData::MeshCards` |
| `TSparseSpanArray<FLumenPrimitiveGroup>` | [[ref_lumen_scene_data]] `FLumenSceneData::PrimitiveGroups` |
| `TSparseSpanArray<FLumenPageTableEntry>` | [[ref_lumen_scene_data]] `FLumenSceneData::PageTable` |
| `FUniqueIndexList` | [[ref_lumen_scene_data]] `CardIndicesToUpdateInBuffer`, `MeshCardsIndicesToUpdateInBuffer` |
| `TSparseUniqueList` | [[ref_lumen_scene_data]] `EvictOldestAllocation()` の DirtyCards 引数 |
