# Serialization API リファレンス（FArchive / FStructuredArchive）

- 上位: [[Serialization/01_overview]]
- 関連: [[Details/a_farchive]] | [[Details/b_asset_serialization]] | [[Details/c_save_game]]
- ソース: `Core/Public/Serialization/Archive.h`, `Core/Public/Serialization/StructuredArchive.h`

---

## FArchive 主要 API

### 状態確認

```cpp
// 方向
bool FArchive::IsLoading() const;    // 読み込みモード
bool FArchive::IsSaving() const;     // 書き込みモード
bool FArchive::IsTransacting() const;// Undo/Redo トランザクション中

// 種別
bool FArchive::IsTextFormat() const;  // テキスト形式（JSON 等）
bool FArchive::IsBinary() const;

// エラー
bool FArchive::IsError() const;
void FArchive::SetError();
void FArchive::ClearError();

// 位置
int64 FArchive::Tell() const;         // 現在の読み書き位置
int64 FArchive::TotalSize() const;    // ストリーム全体のサイズ
void  FArchive::Seek(int64 InPos);    // 位置移動

// バージョン
FPackageFileVersion FArchive::UEVer() const;
int32 FArchive::LicenseeUEVer() const;
FCustomVersion FArchive::GetCustomVersion(FGuid VersionKey) const;
```

### シリアライズ演算子

```cpp
// 全プリミティブ型は operator<< でシリアライズ
FArchive& operator<<(FArchive& Ar, int8&   V);
FArchive& operator<<(FArchive& Ar, int16&  V);
FArchive& operator<<(FArchive& Ar, int32&  V);
FArchive& operator<<(FArchive& Ar, int64&  V);
FArchive& operator<<(FArchive& Ar, uint8&  V);
FArchive& operator<<(FArchive& Ar, uint16& V);
FArchive& operator<<(FArchive& Ar, uint32& V);
FArchive& operator<<(FArchive& Ar, uint64& V);
FArchive& operator<<(FArchive& Ar, float&  V);
FArchive& operator<<(FArchive& Ar, double& V);
FArchive& operator<<(FArchive& Ar, bool&   V);
FArchive& operator<<(FArchive& Ar, FString& V);
FArchive& operator<<(FArchive& Ar, FName&   V);
FArchive& operator<<(FArchive& Ar, UObject*& V);  // UObject 参照

// 生バイト読み書き
void FArchive::Serialize(void* V, int64 Length);

// 配列の一括シリアライズ
template<typename T>
FArchive& operator<<(FArchive& Ar, TArray<T>& A);
```

### カスタムバージョン

```cpp
// バージョン GUID の定義（.cpp に記述）
const FGuid FMyCustomVersion::GUID(0x12345678, 0xABCDABCD, 0x11111111, 0x22222222);
FCustomVersionRegistration GMyCustomVersion(FMyCustomVersion::GUID, FMyCustomVersion::Latest, TEXT("MySystem"));

// UObject::Serialize 内でのバージョン分岐
virtual void Serialize(FArchive& Ar) override
{
    Super::Serialize(Ar);
    Ar.UsingCustomVersion(FMyCustomVersion::GUID);

    int32 CustomVer = Ar.CustomVer(FMyCustomVersion::GUID);
    if (CustomVer < FMyCustomVersion::AddedNewField)
    {
        // 旧バージョンのデフォルト値を設定
        NewField = DefaultValue;
    }
    else
    {
        Ar << NewField;
    }
}
```

---

## メモリアーカイブ

```cpp
// 書き込み
TArray<uint8> Buffer;
FMemoryWriter Writer(Buffer, true);  // bIsPersistent=true
Writer << MyData;

// 読み込み
FMemoryReader Reader(Buffer, true);
Reader << MyData;

// 位置操作
Writer.Seek(0);
int64 Size = Writer.TotalSize();
```

---

## ファイルアーカイブ

```cpp
#include "HAL/FileManager.h"

// 書き込み
FArchive* FileWriter = IFileManager::Get().CreateFileWriter(TEXT("D:/out.bin"));
if (FileWriter)
{
    *FileWriter << MyData;
    FileWriter->Close();
    delete FileWriter;
}

// 読み込み
FArchive* FileReader = IFileManager::Get().CreateFileReader(TEXT("D:/out.bin"));
if (FileReader)
{
    *FileReader << MyData;
    FileReader->Close();
    delete FileReader;
}
```

---

## FObjectAndNameAsStringProxyArchive

`UObject*` 参照をオブジェクト名文字列として保存するプロキシ（SaveGame 内部で使用）:

```cpp
#include "Serialization/ObjectAndNameAsStringProxyArchive.h"

TArray<uint8> Bytes;
FMemoryWriter MemWriter(Bytes);
FObjectAndNameAsStringProxyArchive Proxy(MemWriter, true);

MyObject->Serialize(Proxy);
```

---

## USaveGame API

```cpp
// 生成
USaveGame* UGameplayStatics::CreateSaveGameObject(TSubclassOf<USaveGame> SaveGameClass);

// 同期保存・ロード
bool    UGameplayStatics::SaveGameToSlot(USaveGame* SaveGameObject, const FString& SlotName, int32 UserIndex);
USaveGame* UGameplayStatics::LoadGameFromSlot(const FString& SlotName, int32 UserIndex);

// 存在確認・削除
bool UGameplayStatics::DoesSaveGameExist(const FString& SlotName, int32 UserIndex);
bool UGameplayStatics::DeleteGameInSlot(const FString& SlotName, int32 UserIndex);

// 非同期保存・ロード（UE 5.1 以降推奨）
void UGameplayStatics::AsyncSaveGameToSlot(
    USaveGame* SaveGameObject,
    const FString& SlotName,
    int32 UserIndex,
    FAsyncSaveGameToSlotDelegate SavedDelegate);

void UGameplayStatics::AsyncLoadGameFromSlot(
    const FString& SlotName,
    int32 UserIndex,
    FAsyncLoadGameFromSlotDelegate LoadedDelegate);

// バイナリ変換（メモリ内シリアライズ）
bool UGameplayStatics::SaveGameToMemory(USaveGame* SaveGameObject, TArray<uint8>& OutSaveData);
USaveGame* UGameplayStatics::LoadGameFromMemory(const TArray<uint8>& InSaveData);
```

---

## FLinkerLoad / FLinkerSave（低レベル）

```cpp
// パッケージの非同期ロード（最も一般的な方法）
int32 LoadPackageAsync(
    const FString& InName,
    FLoadPackageAsyncDelegate InCompletionDelegate,
    TAsyncLoadPriority InPackagePriority = 0,
    EPackageFlags InPackageFlags = PKG_None,
    int32 InPIEInstanceID = INDEX_NONE,
    FName InPackagePathName = NAME_None
);

// 同期ロード（非推奨：ゲームスレッドをブロック）
UPackage* LoadPackage(UPackage* InOuter, const TCHAR* InLongPackageName, uint32 LoadFlags);

// パッケージ保存（Editor 専用）
bool UPackage::SavePackage(
    UPackage* InOuter,
    UObject* Base,
    EObjectFlags TopLevelFlags,
    const TCHAR* Filename,
    FOutputDevice* Error = GError,
    FLinker* ConformationLinker = nullptr,
    bool bForceByteSwapping = false,
    bool bWarnOfLongFilename = true,
    uint32 SaveFlags = SAVE_None
);
```

---

## FStructuredArchive（モダン API）

```cpp
#include "Serialization/StructuredArchive.h"

void SerializeStruct(FStructuredArchive::FRecord Record)
{
    Record.GetField(SA_FIELD_NAME(TEXT("Health"))) << Health;
    Record.GetField(SA_FIELD_NAME(TEXT("Name")))   << PlayerName;

    FStructuredArchive::FArray ArraySlot = Record.EnterArray(SA_FIELD_NAME(TEXT("Items")), ItemCount);
    for (int32 i = 0; i < ItemCount; ++i)
    {
        ArraySlot.EnterElement() << Items[i];
    }
}
```

---

## 関連ドキュメント

- [[Details/a_farchive]] — FArchive の基底設計・カスタムバージョンの概念
- [[Details/b_asset_serialization]] — FLinkerLoad/Save・.uasset の内部構造
- [[Details/c_save_game]] — USaveGame の高レベルフロー
