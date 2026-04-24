# 文字列型（FString / FName / FText）

- 上位: [[Containers/01_overview]]
- 関連: [[a_array_map_set]] | [[c_smart_pointers]]
- ソース: `Core/Public/Containers/UnrealString.h`, `Core/Public/UObject/NameTypes.h`, `Core/Public/Internationalization/Text.h`

---

## 3 種類の文字列型

| 型 | 用途 | 変更可 | ローカライズ | メモリ |
|----|------|--------|------------|--------|
| `FString` | 汎用可変文字列 | 可 | 不可 | ヒープ確保 |
| `FName` | ID・識別子（大文字小文字無視） | 不可 | 不可 | グローバル名前テーブル |
| `FText` | 表示用・ローカライズ対応文字列 | 不可 | 可 | ヒープ確保 |

---

## FString

### 基本操作

```cpp
FString Str = TEXT("Hello");

// 連結
FString Joined = Str + TEXT(" World");
Str += TEXT("!");
FString AppendStr = FString::Printf(TEXT("Name: %s, Val: %d"), *Str, 42);

// 長さ
int32 Len = Str.Len();
bool bEmpty = Str.IsEmpty();

// 検索・置換
bool bContains = Str.Contains(TEXT("Hello"));
int32 Idx = Str.Find(TEXT("World"));
Str.ReplaceInline(TEXT("Hello"), TEXT("Goodbye"));
FString Replaced = Str.Replace(TEXT("!"), TEXT("."));

// 変換
FString Upper = Str.ToUpper();
FString Lower = Str.ToLower();
FString Trimmed = Str.TrimStartAndEnd();

// 分割
TArray<FString> Parts;
Str.ParseIntoArray(Parts, TEXT(","));

// サブ文字列
FString Sub = Str.Mid(0, 5);   // 位置0から5文字
FString Left3 = Str.Left(3);
FString Right3 = Str.Right(3);

// 数値変換
int32 IntVal = FCString::Atoi(*Str);
float FloatVal = FCString::Atof(*Str);
FString FromInt = FString::FromInt(42);
FString FromFloat = FString::SanitizeFloat(3.14f);
```

### 文字列リテラル

```cpp
TEXT("...") // FString / FName / FText リテラルには TEXT() マクロを使う
// 内部は wchar_t（Windows）または char16_t（その他）
```

### `*` 演算子（TCHAR へのポインタ）

```cpp
FString MyStr = TEXT("Value");
// *MyStr で const TCHAR* を取得（C API や printf スタイルに渡す）
UE_LOG(LogTemp, Log, TEXT("String: %s"), *MyStr);
```

---

## FName

### 概要と生成

```cpp
FName Name1 = FName(TEXT("MyActor"));
FName Name2 = TEXT("MyActor");   // 暗黙変換
FName None = NAME_None;          // 空の FName

// 大文字小文字無視（同一として扱われる）
FName A = TEXT("hello");
FName B = TEXT("HELLO");
bool bSame = (A == B);   // true
```

### 内部構造

FName はグローバルな名前テーブル（`FNamePool`）にエントリとして登録され、**32 bit の整数インデックス + Number** で表現される:

```
FName:
  ├── ComparisonIndex (int32) … 大文字統一後のエントリ
  ├── DisplayIndex    (int32) … 元の大文字小文字を保持するエントリ
  └── Number          (int32) … "Actor_0" の "_0" 部分
```

### 主要操作

```cpp
FName Name = TEXT("MyObject");

// FString 変換
FString NameStr = Name.ToString();

// 比較
bool bEqual = Name == TEXT("MyObject");
bool bNone = Name.IsNone();

// 有効性
bool bValid = Name.IsValid();
```

FName の比較は O(1)（整数比較）なので、ハッシュマップのキーや識別子に最適。

---

## FText

### 概要

ローカライズに対応した表示用文字列。実際に表示するすべてのテキストに使う。

```cpp
// 直接生成（ローカライズなし。デバッグ用）
FText T1 = FText::FromString(TEXT("Hello"));

// ローカライズキーで生成（推奨）
FText T2 = NSLOCTEXT("MyNamespace", "MyKey", "Default Text");
// または
FText T3 = LOCTEXT("MyKey", "Default Text");  // LOCTEXT_NAMESPACE 定義が必要

// 数値フォーマット
FText IntText = FText::AsNumber(42);
FText FloatText = FText::AsNumber(3.14f);
FText CurrencyText = FText::AsCurrency(100, TEXT("USD"));
FText DateText = FText::AsDateTime(FDateTime::Now());

// フォーマット文字列（{0} プレースホルダ）
FText Formatted = FText::Format(
    NSLOCTEXT("NS", "K", "Score: {0} / {1}"),
    FText::AsNumber(Score), FText::AsNumber(MaxScore)
);

// 比較
bool bSame = T1.EqualTo(T2);
bool bSameCaseInsensitive = T1.EqualToCaseIgnored(T2);

// 内部文字列取得（デバッグ・ログ用）
FString StrVal = T1.ToString();
```

### LOCTEXT_NAMESPACE

```cpp
// .cpp ファイルの先頭と末尾に定義する
#define LOCTEXT_NAMESPACE "MyGame"

FText MyText = LOCTEXT("WelcomeMessage", "Welcome!");

#undef LOCTEXT_NAMESPACE
```

---

## 型変換チートシート

| 変換元 | 変換先 | 方法 |
|--------|--------|------|
| `FString` | `const TCHAR*` | `*MyString` |
| `FString` | `FName` | `FName(*MyString)` |
| `FString` | `FText` | `FText::FromString(MyString)` |
| `FName` | `FString` | `MyName.ToString()` |
| `FName` | `FText` | `FText::FromName(MyName)` |
| `FText` | `FString` | `MyText.ToString()` |
| `int32` | `FString` | `FString::FromInt(N)` または `FString::Printf(TEXT("%d"), N)` |
| `float` | `FString` | `FString::SanitizeFloat(F)` |
| `FString` | `int32` | `FCString::Atoi(*S)` |
| `FString` | `float` | `FCString::Atof(*S)` |
| `std::string` | `FString` | `FString(Str.c_str())` |
| `FString` | `std::string` | `std::string(TCHAR_TO_UTF8(*S))` |

---

## パフォーマンス指針

| 用途 | 推奨型 |
|------|--------|
| Actor・アセット名・Bone 名 | `FName`（比較 O(1)、メモリ効率高） |
| ログ・デバッグ・パス文字列 | `FString`（変更可） |
| UI 表示・ローカライズテキスト | `FText` |
| 内部 ID・enum 替わり | `FName` |
| ハッシュキー | `FName` > `FString` |

`FText` → `FString` 変換は変換コストがあるため、ホットパスで繰り返すのは避ける。

---

## 関連ドキュメント

- [[a_array_map_set]] — TMap<FName, T> など文字列型を使ったコンテナ
- [[c_smart_pointers]] — TSharedPtr<FString> など
- [[Reference/ref_containers_api]] — FString / FName / FText API 全一覧
