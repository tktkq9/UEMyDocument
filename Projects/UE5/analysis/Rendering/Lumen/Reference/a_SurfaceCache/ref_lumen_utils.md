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

**疎な配列 + 安定インデックス + 連続スパン割り当て**のコンテナ。

Card や MeshCards など「インデックスが外部から参照され、途中削除が起きる」データの管理に最適。

```cpp
template <typename ElementType>
class TSparseSpanArray {
public:
    int32 Num() const;                      // 最大サイズ（穴を含む）を返す
    void Reserve(int32 NumElements);        // メモリ予約

    // 連続した NumElements 個のスロットを確保して先頭インデックスを返す
    int32 AddSpan(int32 NumElements);

    // 先頭インデックスと個数を指定してスロットを解放
    void RemoveSpan(int32 FirstElementIndex, int32 NumElements);

    bool IsAllocated(int32 Index) const;    // 指定インデックスが割り当て済みか
    ElementType& operator[](int32 Index);   // 要素アクセス
    const ElementType& operator[](int32 Index) const;
};
```

**使用例（FLumenCard の管理）:**

```
FLumenMeshCards が 3 枚の Card を持つ場合:
  AddSpan(3) → 連続インデックス 10, 11, 12 を確保
  MeshCards.FirstCardIndex = 10
  MeshCards.NumCards = 3

別の MeshCards が削除されても、インデックス 10〜12 は変わらない（安定インデックス）
```

---

## TSparseUniqueList\<ElementType, Allocator\>

**フラット配列 + ハッシュセット** による重複排除リスト。  
重複チェックを O(1) で行いつつ、イテレーションはフラット配列で高速に行える。

```cpp
template <typename ElementType, typename Allocator>
struct TSparseUniqueList {
    // 要素を追加（すでに存在する場合はスキップ）
    void Add(ElementType Element);

    TArray<ElementType, Allocator> Array;             // イテレーション用フラット配列
    Experimental::TSherwoodSet<ElementType> Set;      // 重複チェック用ハッシュセット
};
```

---

## FUniqueIndexList

**フラット配列 + ビットセット** による重複排除インデックスリスト。  
整数インデックスに特化した軽量版。ビットセットで重複チェックを O(1) で行う。

```cpp
struct FUniqueIndexList {
public:
    // インデックスを追加（すでに追加済みならスキップ）
    void Add(int32 Index);

    int32 Num() const;      // 現在の要素数
    void Reset();           // 全リセット（メモリ解放なし）
    void Empty(int32 Slack); // クリア（Slack 分の容量を確保）

    // 範囲 for でイテレーション可能
    auto begin() const;
    auto end() const;

private:
    TArray<int32> Indices;                  // 重複なしのインデックスリスト
    TBitArray<> IndicesMarkedToUpdate;      // 追加済みインデックスのビットセット
};
```

**使用例（更新が必要な Card インデックスの管理）:**

```cpp
FUniqueIndexList DirtyCardIndices;

// 同じ Card が複数回 dirty になっても 1 回だけ更新する
DirtyCardIndices.Add(CardIndex);

// 全 dirty Card を処理
for (int32 CardIndex : DirtyCardIndices) {
    UpdateCard(CardIndex);
}
DirtyCardIndices.Reset();
```

---

## 使用箇所まとめ

| コンテナ | 主な使用箇所 |
|---------|------------|
| `TSparseSpanArray<FLumenCard>` | `FLumenSceneData::Cards` — 全 Card の管理 |
| `TSparseSpanArray<FLumenMeshCards>` | `FLumenSceneData::MeshCards` — 全 MeshCards の管理 |
| `TSparseSpanArray<FLumenPrimitiveGroup>` | `FLumenSceneData::PrimitiveGroups` — 全プリミティブグループ |
| `FUniqueIndexList` | 更新が必要な Card / MeshCards の dirty リスト管理 |
| `TSparseUniqueList` | Ray Tracing グループ等の一意なリスト管理 |
