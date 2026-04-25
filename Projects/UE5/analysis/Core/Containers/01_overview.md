# Containers 概要

- 上位: [[01_core_overview]]
- 関連: [[UObject/01_overview]]
- ソース: `Engine/Source/Runtime/Core/Public/Containers/`, `Engine/Source/Runtime/Core/Public/UObject/NameTypes.h`, `Engine/Source/Runtime/Core/Public/Templates/SharedPointer.h`, `Engine/Source/Runtime/Core/Public/Internationalization/Text.h`

---

## Containers とは

UE5 の **コンテナ・文字列・スマートポインタ** のすべてが含まれる基盤領域。STL とは別に、ゲームエンジン用途に最適化された独自実装を持つ。

大きく 3 カテゴリ:

1. **汎用コンテナ** — `TArray` / `TMap` / `TSet` / `TSparseArray` 等
2. **文字列系** — `FString` / `FName` / `FText` / `FStringView`
3. **スマートポインタ** — `TSharedPtr` / `TWeakPtr` / `TUniquePtr` / `TRefCountPtr`

---

## 汎用コンテナ

### TArray — 最頻出

動的配列。内部は連続メモリ（`std::vector` 相当）:

```cpp
TArray<int32> Arr;
Arr.Add(1);
Arr.Add(2);
Arr.Emplace(3);       // in-place 構築
Arr.Remove(1);        // 値 1 を削除
Arr.RemoveAt(0);      // 先頭を削除
Arr.RemoveAtSwap(0);  // 順序変更許容、O(1) 削除
Arr.Reserve(100);
for (int32 Val : Arr) { ... }
```

| メソッド | 複雑度 | 備考 |
|---------|-------|------|
| `Add(T)` / `Emplace()` | amortized O(1) | 末尾追加 |
| `Insert(T, Index)` | O(N) | 指定位置挿入 |
| `Remove(T)` | O(N) | 値で削除（重複も全部）|
| `RemoveAt(Index)` | O(N) | 順序維持 |
| `RemoveAtSwap(Index)` | O(1) | 末尾要素と入れ替え |
| `Find(T)` | O(N) | 線形検索 |
| `Sort()` | O(N log N) | イントロソート |

### TMap — ハッシュマップ

```cpp
TMap<FString, int32> Scores;
Scores.Add(TEXT("Alice"), 100);
int32* Val = Scores.Find(TEXT("Alice"));
Scores.Remove(TEXT("Bob"));
for (const auto& Pair : Scores) { ... }
```

内部は `TSparseArray<TSet の Pair>` で構成され、`Find` は平均 O(1)。

### TSet — ハッシュセット

```cpp
TSet<int32> Unique;
Unique.Add(42);
bool bContains = Unique.Contains(42);
```

### その他のコンテナ

| 型 | 用途 |
|----|------|
| `TSparseArray<T>` | 穴が開く配列。Index 再利用せず |
| `TBitArray` | ビット単位の配列 |
| `TStaticArray<T, N>` | サイズ固定（スタック確保可） |
| `TQueue<T>` | ロックフリー SPSC キュー |
| `TDeque<T>` | 両端キュー |
| `TRingBuffer<T>` | 循環バッファ |
| `TMpscQueue<T>` | マルチプロデューサ/シングルコンシューマ |
| `TArrayView<T>` | 非所有ビュー（ポインタ + 長さ） |
| `TStridedView<T>` | ストライド付きビュー |
| `TSortedMap<K, V>` | 整列マップ（ツリーベース） |

---

## 文字列系

UE5 には用途別に **3 種類の文字列型** がある:

| 型 | ストレージ | 変更可 | 比較速度 | 用途 |
|----|-----------|------|---------|------|
| `FString` | `TArray<TCHAR>` | Yes | 遅い（O(N)） | 汎用文字列。ログ・パス・一般処理 |
| `FName` | 共有プール | No | **O(1)**（index 比較） | アセット名・識別子・タグ |
| `FText` | ローカライズ対応ストレージ | No | 概念的に遅い | UI 表示用（翻訳される） |

### FString

```cpp
FString Str = TEXT("Hello");
Str += TEXT(", World");
int32 Len = Str.Len();
FString Upper = Str.ToUpper();
bool bEqual = Str == TEXT("Hello, World");
```

### FName

内部は `FNameEntry` のグローバルプール。同じ文字列は同じ `FName` インデックスを共有:

```cpp
FName Tag(TEXT("Player"));
if (Actor->Tags.Contains(Tag)) { ... }   // O(1) 比較
FString Str = Tag.ToString();            // 逆変換
```

**大文字小文字は区別しない**（デフォルト）。

### FText

```cpp
FText Greeting = FText::FromString(TEXT("Hello"));
FText Localized = NSLOCTEXT("Game", "Greeting", "Hello");  // ローカライズキー
FText Formatted = FText::Format(NSLOCTEXT("Game", "Score", "Score: {0}"),
    FText::AsNumber(100));
```

StringTable と連動してランタイム翻訳可能。

### FStringView — 非所有

```cpp
FStringView View = TEXTVIEW("Hello");
void LogMessage(FStringView Msg);  // 引数で渡すとき高速
```

---

## スマートポインタ（非 UObject）

**UObject は GC で管理されるため、スマートポインタは原則 非 UObject にのみ使う**（UObject には `TObjectPtr`/`TWeakObjectPtr` を使う）。

| 型 | 所有権 | 備考 |
|----|-------|------|
| `TSharedPtr<T>` | 共有（nullable） | 参照カウント |
| `TSharedRef<T>` | 共有（非 null 保証） | `nullptr` 許容せず |
| `TWeakPtr<T>` | 弱参照 | 破棄検出可 |
| `TUniquePtr<T>` | 単独（移動のみ） | `std::unique_ptr` 相当 |
| `TRefCountPtr<T>` | 侵入型参照カウント | オブジェクトが `FRefCountedObject` 派生 |

### スレッド安全版

```cpp
TSharedPtr<T, ESPMode::ThreadSafe> Ptr;      // 原子的な参照カウント
TSharedRef<T, ESPMode::ThreadSafe> Ref;
```

### 作成

```cpp
TSharedPtr<FMyStruct> Ptr = MakeShared<FMyStruct>(Arg1, Arg2);
TSharedRef<FMyStruct> Ref = MakeShared<FMyStruct>();
TUniquePtr<FMyStruct> UPtr = MakeUnique<FMyStruct>();
```

---

## 主要クラス

| クラス | 役割 |
|-------|------|
| `TArray<T, Allocator>` | 可変長配列。Allocator でアロケータ切替可 |
| `TMap<K, V>` | ハッシュマップ |
| `TSet<T>` | ハッシュセット |
| `TSparseArray<T>` | 穴あき配列 |
| `FString` | 可変長文字列 |
| `FStringView` | 非所有文字列ビュー |
| `FName` | 共有プール文字列 |
| `FText` | ローカライズ対応文字列 |
| `TSharedPtr<T>` / `TSharedRef<T>` / `TWeakPtr<T>` | 共有所有権 |
| `TUniquePtr<T>` | 単独所有権 |
| `TRefCountPtr<T>` | 侵入型参照カウント |
| `FArchiveContainerElement` | コンテナのシリアライズ支援 |

---

## Details（個別記事）

| ドキュメント | 内容 |
|------------|------|
| [[Details/a_array_map_set]] | `TArray`/`TMap`/`TSet`/`TSparseArray` の内部構造・アロケータ・反復子 |
| [[Details/b_string_types]] | `FString`/`FName`/`FText` の使い分け・変換・ローカライズ |
| [[Details/c_smart_pointers]] | `TSharedPtr`/`TWeakPtr`/`TUniquePtr`/`TSharedRef`・スレッド安全版 |

---

## Reference

- [[Reference/ref_containers_api]] … `TArray` / `TMap` / `TSet` / `FString` / `TSharedPtr` 主要メソッド

---

## 使い分けガイド

| 用途 | 推奨型 |
|------|-------|
| 一般的な配列 | `TArray` |
| ルックアップが必要な辞書 | `TMap` |
| 重複排除したい集合 | `TSet` |
| インデックスを保持したまま要素削除 | `TSparseArray` |
| スレッド間でキュー | `TQueue`（SPSC）/ `TMpscQueue`（MPSC） |
| ログ・パス・汎用文字列 | `FString` |
| アセット名・タグ・識別子 | `FName` |
| UI に表示する翻訳対象文字列 | `FText` |
| 関数引数の文字列（変更しない） | `FStringView` |
| 非 UObject の共有所有 | `TSharedPtr` |
| 非 UObject の単独所有 | `TUniquePtr` |
| UObject の参照 | `TObjectPtr`（UPROPERTY 内）/ `TWeakObjectPtr`（破棄検出可） |

---

## コード実行フロー

### エントリポイント（コンテナ操作・文字列・スマートポインタ）

```
(TArray)
TArray::Add(Element)                                              [Array.h]
  └─ TArray::AddUninitialized(1)
       ├─ ResizeAllocation(NewMax) (要素数 == 容量なら)            ← ヒューリスティックで Grow
       │    └─ Allocator::ResizeAllocation()                       ← FHeapAllocator が realloc
       └─ new (GetData() + Index) ElementType(Element)             ← placement new

(TMap)
TMap::Add(Key, Value)                                             [Map.h]
  └─ TSortableMapBase::Emplace()                                   [Map.h]
       └─ Pairs.Emplace(MoveTemp(Key), MoveTemp(Value))             ← TSet<TPair> に格納
            └─ TSet::Emplace(KeyHash, Pair)                         ← ハッシュ計算 + バケット解決

(FName)
FName Name(TEXT("Player"))                                         [NameTypes.h]
  └─ FName::Init(WideName, FNAME_Add)
       ├─ FNameHelper::CalculateHash(Str)                           ← ハッシュ計算
       ├─ FNamePool::Find(Hash, Str)                                ← グローバルプール検索
       │    └─ ヒット: 既存 EntryId 返却                            ← O(1) 比較に貢献
       └─ Miss: FNameEntry をプールに追加 + EntryId 払い出し

(FString)
FString Str = TEXT("Hello") + FString(TEXT(", World"));            [String.h]
  └─ FString::operator+ → FString::operator+=
       └─ Data.Append(Other.Data)                                   ← 内部は TArray<TCHAR>

(TSharedPtr)
MakeShared<FFoo>(Args...)                                          [SharedPointer.h]
  └─ new TIntrusiveReferenceController<FFoo>(...)                  ← オブジェクト + 参照カウンタ統合確保
       └─ TSharedPtr<FFoo>(Controller)                              ← SharedReferenceCount = 1

TSharedPtr ~ destructor
  └─ TReferenceControllerBase::ReleaseSharedReference()            ← Atomic デクリメント
       └─ if (SharedRefCount == 0) DestroyObject()                  ← 0 で破棄
            └─ if (WeakRefCount == 0) delete Controller             ← 弱参照も尽きたら本体解放
```

### フロー詳細

1. **TArray 拡張** — `Add` が容量不足を検知すると `ResizeAllocation` が `Allocator` に再確保を依頼。標準は `FHeapAllocator` で `FMemory::Realloc` を呼ぶ。Grow 戦略は `DefaultCalculateSlackGrow`（[[Details/a_array_map_set]]）。
2. **TMap/TSet 内部** — `TMap` は `TSet<TPair<K,V>>` の薄いラッパ。`TSet` は `TSparseArray` + ハッシュバケット（`TArray<int32>`）の二段構成で、要素は穴あき配列に格納される。
3. **FName プール** — グローバル `FNamePool` がスレッドセーフに `FNameEntry` を管理。同一文字列は単一エントリを共有し、`FName` 比較は EntryId 比較で O(1)（[[Details/b_string_types]]）。
4. **FString 操作** — 内部は `TArray<TCHAR>` のため、Append/Concat は TArray と同じ Grow ロジック。`FString::Printf` は可変長書式化を行う。
5. **FText ローカライズ** — `NSLOCTEXT` マクロが `FTextHistory` にローカライズキーを記録。実表示時に `FInternationalization` が翻訳を解決。
6. **SharedPtr 参照カウント** — `MakeShared` は `TIntrusiveReferenceController` を 1 アロケーションで生成（オブジェクト本体と参照カウンタを連結）。`ESPMode::ThreadSafe` 指定で参照カウントが Atomic 化される（[[Details/c_smart_pointers]]）。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `TArray::Add` / `ResizeAllocation` | `Array.h` | 動的配列の追加・再確保 |
| `FHeapAllocator::ResizeAllocation` | `ContainerAllocationPolicies.h` | デフォルトアロケータ |
| `TSet::Emplace` | `Set.h` | ハッシュテーブルへの挿入 |
| `FNamePool::Find` / `Store` | `NameTypes.cpp` | FName プール管理 |
| `FString::operator+=` | `String.h` | 文字列連結 |
| `TIntrusiveReferenceController` | `SharedPointerInternals.h` | SharedPtr 参照カウンタ |
| `MakeShared<T>` | `SharedPointer.h` | SharedPtr 生成（1 アロック） |

---

## 備考

- **TArray の削除順序**: 順序維持なら `RemoveAt`、気にしないなら `RemoveAtSwap`（O(1)）
- **FName は大文字小文字を無視** — 比較は case-insensitive（ただし表示文字列は初回登録時のまま）
- **FString への演算子**: `+`/`+=` で連結、`FString::Printf(TEXT("..."), ...)` で書式化
- **スマートポインタは GC 非対応** — UObject には絶対使わないこと
- **スレッドセーフ SharedPtr** — `ESPMode::ThreadSafe` 指定。Atomic な参照カウントのため非スレッドセーフ版より遅い
