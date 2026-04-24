# TArray / TMap / TSet / FString / FName / FText API リファレンス

- 上位: [[Containers/01_overview]]
- 関連: [[Details/a_array_map_set]] | [[Details/b_string_types]] | [[Details/c_smart_pointers]]
- ソース: `Core/Public/Containers/Array.h`, `Core/Public/Containers/Map.h`, `Core/Public/Containers/Set.h`

---

## TArray\<T\>

### 追加・挿入

```cpp
int32 Add(const T& Item);           // 末尾に追加・インデックスを返す
int32 Add(T&& Item);                // ムーブ版
int32 AddUnique(const T& Item);     // 重複なければ追加（なければ INDEX_NONE）
int32 Emplace(ArgsType&&... Args);  // 直接構築して追加
void  Insert(const T& Item, int32 Index);
void  Append(const TArray<T>& Other);
void  Append(TArray<T>&& Other);
```

### アクセス

```cpp
T&       operator[](int32 Index);
const T& operator[](int32 Index) const;
T&       Last(int32 IndexFromEnd = 0);
const T& Last(int32 IndexFromEnd = 0) const;
T*       GetData();
int32    Num() const;
int32    Max() const;               // 確保済み容量
bool     IsEmpty() const;
bool     IsValidIndex(int32 Index) const;
SIZE_T   GetAllocatedSize() const;
```

### 検索

```cpp
int32 Find(const T& Item) const;                                   // 前から検索（なければ INDEX_NONE）
int32 FindLast(const T& Item) const;                               // 後ろから検索
bool  Find(const T& Item, int32& Index) const;
bool  Contains(const T& Item) const;
bool  ContainsByPredicate(TFunction<bool(const T&)> Pred) const;
T*    FindByKey(const KeyType& Key) const;                         // TArray<TPair> 等
T*    FindByPredicate(TFunction<bool(const T&)> Pred) const;
int32 IndexOfByKey(const KeyType& Key) const;
int32 IndexOfByPredicate(TFunction<bool(const T&)> Pred) const;
```

### 削除

```cpp
int32 Remove(const T& Item);                  // 全一致を削除（削除数を返す）
int32 RemoveSingle(const T& Item);            // 最初の一致のみ削除
void  RemoveAt(int32 Index, int32 Count = 1, bool bAllowShrinking = true);
void  RemoveAtSwap(int32 Index, int32 Count = 1, bool bAllowShrinking = true);
int32 RemoveAll(TFunction<bool(const T&)> Pred);   // 条件で全削除
int32 RemoveAllSwap(TFunction<bool(const T&)> Pred);
void  Pop(bool bAllowShrinking = true);
T     PopValue(bool bAllowShrinking = true);
void  Empty(int32 Slack = 0);
void  Reset(int32 NewSize = 0);
```

### サイズ管理

```cpp
void  Reserve(int32 Number);         // メモリ予約（要素数は変わらない）
void  SetNum(int32 NewNum, bool bAllowShrinking = true);
void  SetNumZeroed(int32 NewNum, bool bAllowShrinking = true);
void  SetNumUninitialized(int32 NewNum, bool bAllowShrinking = true);
void  Shrink();
void  Init(const T& Element, int32 Number);  // 全要素を Element で初期化
```

### ソート

```cpp
void Sort();
void Sort(TFunction<bool(const T&, const T&)> Pred);
void StableSort();
void StableSort(TFunction<bool(const T&, const T&)> Pred);
void HeapSort();
```

### イテレーション

```cpp
for (T& Item : Array) { }
for (const T& Item : Array) { }

// 削除しながらのイテレーション
for (int32 i = Array.Num() - 1; i >= 0; --i)
{
    if (ShouldRemove(Array[i])) Array.RemoveAt(i);
}
```

---

## TMap\<K, V\>

```cpp
// 追加・更新
V&    Add(const K& Key, const V& Value);
V&    Add(K&& Key, V&& Value);
V&    FindOrAdd(const K& Key);           // なければデフォルト挿入
V&    Emplace(K&& Key, V&& Value);
void  Append(const TMap<K, V>& Other);

// アクセス
V*    Find(const K& Key);               // なければ nullptr
const V* Find(const K& Key) const;
V&    FindChecked(const K& Key);        // なければアサート
V&    operator[](const K& Key);
const V& operator[](const K& Key) const;
V     FindRef(const K& Key) const;      // なければデフォルト値

// 確認・削除
bool  Contains(const K& Key) const;
int32 Remove(const K& Key);
void  Empty(int32 Slack = 0);
void  Reset();
int32 Num() const;
bool  IsEmpty() const;

// イテレーション
for (auto& Pair : Map) { K Key = Pair.Key; V Val = Pair.Value; }
for (auto It = Map.CreateIterator(); It; ++It) { It.RemoveCurrent(); }

// キー・バリュー取得
TArray<K> Keys;   Map.GetKeys(Keys);
TArray<V> Values; Map.GenerateValueArray(Values);
TSet<K>   KeySet = Map.KeySet();

// サイズ管理
void Reserve(int32 Number);
void Shrink();
void Compact();
```

---

## TSet\<T\>

```cpp
// 追加
FSetElementId Add(const T& Item);        // FSetElementId を返す
FSetElementId Add(T&& Item);
FSetElementId Emplace(ArgsType&&... Args);

// 確認・検索
bool Contains(const T& Item) const;
T*   Find(const T& Item);
const T* Find(const T& Item) const;

// 削除
bool Remove(const T& Item);             // 削除できたか返す
void Empty(int32 Slack = 0);
void Reset();

// 集合演算
TSet<T> Union(const TSet<T>& Other) const;
TSet<T> Intersect(const TSet<T>& Other) const;
TSet<T> Difference(const TSet<T>& Other) const;
bool    Includes(const TSet<T>& Other) const;

// 状態
int32 Num() const;
bool  IsEmpty() const;
```

---

## FString

```cpp
// 生成
FString FString::Printf(const TCHAR* Fmt, ...);
FString FString::FromInt(int32 V);
FString FString::SanitizeFloat(double V);
FString FString::Chr(TCHAR Ch);
FString FString::ChrN(int32 NumChar, TCHAR Ch);

// 変換
const TCHAR* operator*() const;   // *Str で TCHAR* を取得
FString      ToUpper() const;
FString      ToLower() const;
FString      TrimStart() const;
FString      TrimEnd() const;
FString      TrimStartAndEnd() const;
FString      TrimChar(TCHAR Char) const;

// 検索
bool    Contains(const TCHAR* SubStr, ESearchCase::Type = ESearchCase::IgnoreCase) const;
int32   Find(const TCHAR* SubStr, ESearchCase::Type = ESearchCase::IgnoreCase, ESearchDir::Type = ESearchDir::FromStart, int32 StartPos = INDEX_NONE) const;
bool    StartsWith(const TCHAR* InPrefix, ESearchCase::Type = ESearchCase::IgnoreCase) const;
bool    EndsWith(const TCHAR* InSuffix, ESearchCase::Type = ESearchCase::IgnoreCase) const;
bool    MatchesWildcard(const TCHAR* Wildcard, ESearchCase::Type = ESearchCase::IgnoreCase) const;

// 編集
FString& ReplaceInline(const TCHAR* SearchText, const TCHAR* ReplacementText);
FString  Replace(const TCHAR* SearchText, const TCHAR* ReplacementText) const;
void     ParseIntoArray(TArray<FString>& OutArray, const TCHAR* Delim) const;
void     ParseIntoArrayLines(TArray<FString>& OutArray) const;
FString  Join(const TArray<FString>& Arr, const TCHAR* Separator);

// サブ文字列
FString  Mid(int32 Start, int32 Count = INT32_MAX) const;
FString  Left(int32 Count) const;
FString  Right(int32 Count) const;
FString  LeftChop(int32 Count) const;
FString  RightChop(int32 Count) const;

// 比較
bool     Equals(const FString& Other, ESearchCase::Type = ESearchCase::CaseSensitive) const;
int32    Compare(const FString& Other, ESearchCase::Type = ESearchCase::CaseSensitive) const;

// 状態
int32    Len() const;
bool     IsEmpty() const;
bool     IsNumeric() const;
```

---

## FName

```cpp
// 生成
FName NAME_None;
FName Name(const TCHAR* Str);
FName Name(FString Str);
FName Name = TEXT("Name");  // 暗黙変換

// 変換
FString  ToString() const;
void     ToString(FString& Out) const;
FString  GetPlainNameString() const;  // Number 部分を除いた名前
int32    GetNumber() const;           // "Actor_0" の 0 部分

// 比較（大文字小文字無視）
bool operator==(const FName& Other) const;
bool operator!=(const FName& Other) const;
bool IsNone() const;
bool IsValid() const;

// FName → FText
FText FText::FromName(FName Name);
```

---

## FText

```cpp
// 生成
FText FText::FromString(const FString& S);   // ローカライズなし（デバッグ用）
FText FText::FromName(FName Name);
FText FText::GetEmpty();

// フォーマット
FText FText::Format(FTextFormat Fmt, ...);
FText FText::AsNumber(T Value, const FNumberFormattingOptions* = nullptr);
FText FText::AsCurrency(T Value, const FString& CurrencyCode, const FNumberFormattingOptions* = nullptr);
FText FText::AsPercent(double Value, const FNumberFormattingOptions* = nullptr);
FText FText::AsDateTime(const FDateTime& DateTime, EDateTimeStyle::Type DateStyle = ..., const FString& TimeZone = ..., const FCulturePtr& Culture = nullptr);
FText FText::AsDate(const FDateTime& DateTime, EDateTimeStyle::Type DateStyle = ..., const FString& TimeZone = ..., const FCulturePtr& Culture = nullptr);
FText FText::AsMemory(uint64 NumBytes, const FNumberFormattingOptions* = nullptr, const FCulturePtr& = nullptr, EMemoryUnitStandard::Type = SI);

// 比較
bool EqualTo(const FText& Other, ETextComparisonLevel::Type ComparisonLevel = ETextComparisonLevel::Default) const;
bool EqualToCaseIgnored(const FText& Other) const;
bool IsEmpty() const;
bool IsEmptyOrWhitespace() const;

// 変換
FString ToString() const;
```

---

## スマートポインタ API

```cpp
// TSharedPtr<T>
TSharedPtr<T> MakeShared<T>(Args...);                    // 推奨（1回のアロケーション）
TSharedPtr<T> MakeShareable(T* RawPtr);                  // 生ポインタから生成
TSharedPtr<T> StaticCastSharedPtr<T>(TSharedPtr<From>);  // 静的キャスト
TSharedPtr<T> ConstCastSharedPtr<T>(TSharedPtr<From>);   // const キャスト

T*   Get() const;
T&   operator*() const;
T*   operator->() const;
bool IsValid() const;
void Reset();
int32 GetSharedReferenceCount() const;
TSharedRef<T> ToSharedRef() const;   // null なら assert

// TWeakPtr<T>
TSharedPtr<T> Pin() const;     // ロック（失効していれば null を返す）
bool IsValid() const;
void Reset();

// TUniquePtr<T>
TUniquePtr<T> MakeUnique<T>(Args...);
T*   Get() const;
T*   Release();                // 所有権を放棄
void Reset(T* InPtr = nullptr);
```

---

## 関連ドキュメント

- [[Details/a_array_map_set]] — TArray/TMap/TSet の概念・計算量
- [[Details/b_string_types]] — FString/FName/FText の変換チートシート
- [[Details/c_smart_pointers]] — スマートポインタの選択ガイド
