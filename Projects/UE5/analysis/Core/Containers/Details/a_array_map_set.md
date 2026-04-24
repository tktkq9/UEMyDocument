# TArray / TMap / TSet

- 上位: [[Containers/01_overview]]
- 関連: [[b_string_types]] | [[c_smart_pointers]]
- ソース: `Core/Public/Containers/Array.h`, `Core/Public/Containers/Map.h`, `Core/Public/Containers/Set.h`

---

## TArray

### 基本操作

```cpp
TArray<int32> Arr;

// 追加
Arr.Add(1);           // 末尾に追加
Arr.AddUnique(2);     // 重複しなければ追加
Arr.Insert(5, 0);     // インデックス 0 に挿入
Arr.Emplace(3);       // 直接構築（コピーコスト削減）

// アクセス
int32 Val = Arr[0];
int32& Ref = Arr.Last();       // 末尾要素の参照
int32* Ptr = Arr.GetData();    // 生ポインタ（連続メモリ保証）

// サイズ
int32 Num = Arr.Num();
bool bEmpty = Arr.IsEmpty();

// 検索
int32 Idx = Arr.Find(5);        // なければ INDEX_NONE
bool bContains = Arr.Contains(2);
int32* Found = Arr.FindByPredicate([](int32 V) { return V > 3; });

// 削除
Arr.Remove(1);                   // 値で削除（全一致を削除）
Arr.RemoveAt(0);                 // インデックスで削除（順序維持）
Arr.RemoveAtSwap(0);             // インデックスで削除（末尾と交換→高速）
Arr.RemoveAll([](int32 V) { return V < 0; });  // 条件で削除

// ソート
Arr.Sort();                      // デフォルト昇順
Arr.Sort([](const int32& A, const int32& B) { return A > B; });  // 降順
Arr.StableSort(...);             // 安定ソート

// その他
Arr.Reserve(100);                // メモリ予約（再確保を防ぐ）
Arr.SetNum(50);                  // 要素数を設定（不足分はゼロ初期化）
Arr.Empty();                     // 全削除（メモリは保持）
Arr.Reset();                     // 全削除（Empty と同じ）
Arr.Shrink();                    // 未使用メモリを解放
```

### メモリレイアウト

TArray は **連続メモリ**（`std::vector` 相当）。要素は `TAllocator`（デフォルト: `FDefaultAllocator`）で管理される。

```
[Capacity] [Num]
[E0][E1][E2][E3]...[未使用領域]
```

- `Num()`: 使用中の要素数
- `Max()` / `GetAllocatedSize()`: 確保済みの容量

---

## TMap

キーと値のペアを管理するハッシュマップ（`std::unordered_map` 相当）。

```cpp
TMap<FString, int32> ScoreMap;

// 追加 / 更新
ScoreMap.Add(TEXT("Player1"), 100);
ScoreMap.Emplace(TEXT("Player2"), 200);
ScoreMap.FindOrAdd(TEXT("Player3")) = 150;  // なければデフォルト挿入して参照を返す

// アクセス
int32* Score = ScoreMap.Find(TEXT("Player1"));   // なければ nullptr
int32& Score2 = ScoreMap[TEXT("Player1")];       // なければアサート

// 存在確認
bool bExists = ScoreMap.Contains(TEXT("Player1"));

// 削除
ScoreMap.Remove(TEXT("Player1"));

// 全要素イテレート
for (auto& Pair : ScoreMap)
{
    FString Key   = Pair.Key;
    int32   Value = Pair.Value;
}

// キー / バリューのみ
TArray<FString> Keys;
ScoreMap.GetKeys(Keys);
TArray<int32> Values;
ScoreMap.GenerateValueArray(Values);

// サイズ
int32 Num = ScoreMap.Num();
ScoreMap.Empty();
ScoreMap.Reserve(50);
```

### TSortedMap（ソート済みマップ）

順序が必要な場合:

```cpp
TSortedMap<int32, FString> SortedMap;
SortedMap.Add(3, TEXT("C"));
SortedMap.Add(1, TEXT("A"));
// イテレートはキー昇順で行われる
```

---

## TSet

重複なし要素のハッシュセット（`std::unordered_set` 相当）。

```cpp
TSet<FName> VisitedNames;

// 追加
VisitedNames.Add(FName(TEXT("Level1")));
VisitedNames.Add(FName(TEXT("Level1")));  // 重複は無視される

// 存在確認
bool bVisited = VisitedNames.Contains(FName(TEXT("Level1")));

// 削除
VisitedNames.Remove(FName(TEXT("Level1")));

// イテレート
for (const FName& Name : VisitedNames)
{
    // 順序不定
}

// 集合演算
TSet<FName> SetA, SetB;
TSet<FName> Union     = SetA.Union(SetB);        // 和集合
TSet<FName> Intersect = SetA.Intersect(SetB);    // 積集合
TSet<FName> Diff      = SetA.Difference(SetB);   // 差集合
```

---

## TSparseArray — スパース配列

要素が中間削除されても**インデックスを再利用**するコンテナ。`TMap` のキーが `int32` になったイメージ:

```cpp
TSparseArray<FMyData> SparseArr;

// 追加（返されるインデックスは再利用される）
int32 Idx = SparseArr.Add(FMyData{});

// インデックスアクセス（O(1)）
FMyData& Data = SparseArr[Idx];

// 有効性確認
bool bValid = SparseArr.IsValidIndex(Idx);

// 削除（インデックスは空きスロットとして記録）
SparseArr.RemoveAt(Idx);
```

Actor/Component のスポーン管理など、ID が安定している必要がある場面に使われる。

---

## コンテナの計算量比較

| 操作 | TArray | TMap | TSet |
|------|--------|------|------|
| インデックスアクセス | O(1) | - | - |
| 検索（値） | O(n) | O(1) 平均 | O(1) 平均 |
| 追加 | O(1) 償却 | O(1) 平均 | O(1) 平均 |
| 削除（先頭） | O(n) | O(1) 平均 | O(1) 平均 |
| 削除（末尾）| O(1) | - | - |
| ソート | O(n log n) | - | - |
| メモリ効率 | 高（連続）| 中 | 中 |

---

## UPROPERTY との組み合わせ

`UPROPERTY` で公開するとエディタ・BP・GC 対象になる:

```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite)
TArray<AActor*> ActorList;   // GC がポインタを追跡

UPROPERTY(EditAnywhere)
TMap<FName, int32> StatsMap;

UPROPERTY(EditAnywhere)
TSet<UObject*> ObjectSet;
```

---

## 関連ドキュメント

- [[b_string_types]] — FString / FName / FText
- [[c_smart_pointers]] — TArray と組み合わせる TSharedPtr / TWeakPtr
- [[Reference/ref_containers_api]] — TArray / TMap / TSet API 全一覧
