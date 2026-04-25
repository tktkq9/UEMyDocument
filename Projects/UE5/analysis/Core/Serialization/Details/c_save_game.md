# SaveGame システム

- 上位: [[Serialization/01_overview]]
- 関連: [[a_farchive]] | [[b_asset_serialization]]
- ソース: `Engine/Classes/GameFramework/SaveGame.h`, `Engine/Classes/Kismet/GameplayStatics.h`

---

## 概要

UE5 の **SaveGame** システムはプレイヤーのセーブデータ（進捗・設定・スコア等）を管理する高レベル API。`USaveGame` 派生クラスに `UPROPERTY(SaveGame)` を付けたメンバを定義し、`UGameplayStatics` の関数でスロット名を指定して保存・ロードする。内部は `FMemoryWriter`/`FMemoryReader` を通じた `FArchive` ベースのバイナリ形式。

---

## 基本フロー

```mermaid
graph TD
    DEF[USaveGame 派生クラスを定義<br>UPROPERTY SaveGame を付加] --> CREATE
    CREATE[CreateSaveGameObject T] --> FILL[データをセット]
    FILL --> SAVE[SaveGameToSlot<br>SlotName UserIndex]
    SAVE --> DISK[ディスク / プラットフォームストレージ]

    DISK --> LOAD[LoadGameFromSlot<br>SlotName UserIndex]
    LOAD --> CAST[Cast to T]
    CAST --> READ[データを読み出して使用]
```

---

## 実装手順

### 1. SaveGame クラス定義

```cpp
UCLASS()
class UMySaveGame : public USaveGame
{
    GENERATED_BODY()

public:
    UPROPERTY(SaveGame)
    int32 PlayerLevel = 1;

    UPROPERTY(SaveGame)
    float PlayTimeHours = 0.f;

    UPROPERTY(SaveGame)
    FString PlayerName = TEXT("Unknown");

    UPROPERTY(SaveGame)
    TArray<FString> UnlockedItems;

    // SaveGame が付いていないメンバはシリアライズされない
    int32 RuntimeOnlyData = 0;  // 保存されない
};
```

`UPROPERTY(SaveGame)` はフラグ `CPF_SaveGame` を立てる。`FArchive::IsSavingObjectFlags()` がこのフラグを持つプロパティだけを書く。

### 2. セーブ

```cpp
// C++
UMySaveGame* SaveObj = Cast<UMySaveGame>(
    UGameplayStatics::CreateSaveGameObject(UMySaveGame::StaticClass()));
SaveObj->PlayerLevel = 5;
SaveObj->PlayerName = TEXT("Player1");

bool bSuccess = UGameplayStatics::SaveGameToSlot(SaveObj, TEXT("Slot1"), 0);
```

```cpp
// Blueprint: SaveGameToSlot ノード（スロット名・ユーザーIndex を入力）
```

### 3. ロード

```cpp
// C++
USaveGame* RawSave = UGameplayStatics::LoadGameFromSlot(TEXT("Slot1"), 0);
UMySaveGame* SaveObj = Cast<UMySaveGame>(RawSave);
if (SaveObj)
{
    int32 Level = SaveObj->PlayerLevel;  // 復元された値
}
```

### 4. スロット存在確認と削除

```cpp
bool bExists = UGameplayStatics::DoesSaveGameExist(TEXT("Slot1"), 0);
UGameplayStatics::DeleteGameInSlot(TEXT("Slot1"), 0);
```

---

## 非同期保存・ロード

ゲームスレッドをブロックしない推奨方式（UE 5.1 以降）:

```cpp
// 非同期セーブ
UGameplayStatics::AsyncSaveGameToSlot(
    SaveObj,
    TEXT("Slot1"),
    0,
    FAsyncSaveGameToSlotDelegate::CreateLambda(
        [](const FString& SlotName, const int32 UserIndex, bool bSuccess) {
            if (bSuccess) { /* 成功処理 */ }
        }));

// 非同期ロード
UGameplayStatics::AsyncLoadGameFromSlot(
    TEXT("Slot1"),
    0,
    FAsyncLoadGameFromSlotDelegate::CreateLambda(
        [](const FString& SlotName, const int32 UserIndex, USaveGame* SaveGame) {
            UMySaveGame* Save = Cast<UMySaveGame>(SaveGame);
            if (Save) { /* 使用 */ }
        }));
```

---

## 内部シリアライズフロー

`SaveGameToSlot` 内部の動作:

```
SaveGameToSlot(SaveGame, SlotName, UserIndex)
  ├─ UGameplayStatics::SaveGameToMemory(SaveGame)
  │    ├─ TArray<uint8> Bytes を確保
  │    ├─ FMemoryWriter MemWriter(Bytes)
  │    ├─ SaveGame ヘッダ書き込み（クラス名・バージョン）
  │    └─ FObjectAndNameAsStringProxyArchive ProxyAr(MemWriter)
  │         └─ SaveGame->Serialize(ProxyAr)
  │              → CPF_SaveGame フラグのプロパティのみ書く
  │
  └─ ISaveGameSystem::SaveGame(SlotName, UserIndex, Bytes)
       → プラットフォーム固有の書き込み（Windows: ファイル、Console: 専用ストレージ）
```

`FObjectAndNameAsStringProxyArchive` を使うことで、`UObject*` 参照をオブジェクト名文字列として保存（外部参照が壊れないようにするため）。

---

## バージョン管理

SaveGame は `FCustomVersion` でのバージョン管理を推奨:

```cpp
UCLASS()
class UMySaveGame : public USaveGame
{
    GENERATED_BODY()

    UPROPERTY(SaveGame)
    int32 SaveVersion = 0;  // 独自バージョンフィールド

    virtual void Serialize(FArchive& Ar) override
    {
        Super::Serialize(Ar);
        if (Ar.IsLoading() && SaveVersion < 2)
        {
            // バージョン 2 以前は PlayerName が存在しなかった
            PlayerName = TEXT("Legacy Player");
        }
    }
};
```

---

## JSON形式での保存（手動実装）

エンジン標準はバイナリ。JSON での保存はカスタム実装が必要:

```cpp
// UObject → JSON
TSharedPtr<FJsonObject> JsonObj = MakeShared<FJsonObject>();
for (TFieldIterator<FProperty> It(MySave->GetClass()); It; ++It)
{
    FProperty* Prop = *It;
    if (Prop->HasAnyPropertyFlags(CPF_SaveGame))
    {
        TSharedPtr<FJsonValue> JsonVal;
        FJsonObjectConverter::UPropertyToJsonValue(Prop, PropPtr, JsonVal, 0, CPF_SaveGame);
        JsonObj->SetField(Prop->GetName(), JsonVal);
    }
}
FString JsonStr;
TSharedRef<TJsonWriter<>> Writer = TJsonWriterFactory<>::Create(&JsonStr);
FJsonSerializer::Serialize(JsonObj.ToSharedRef(), Writer);
```

または `FStructuredArchive` + `JsonArchiveOutputFormatter` を利用。

---

## プラットフォーム別ストレージパス

| プラットフォーム | パス |
|----------------|------|
| Windows | `%LOCALAPPDATA%/[Project]/Saved/SaveGames/[SlotName].sav` |
| Mac | `~/Library/Application Support/[Project]/Saved/SaveGames/` |
| PlayStation | ユーザー固有の Protected Storage |
| Xbox | Title Storage / Connected Storage |

`ISaveGameSystem` がプラットフォーム差を抽象化。`FGenericSaveGameSystem`（ファイルベース）が非コンソールのデフォルト実装。

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `SaveGame.AsyncSave` | `1` | 非同期セーブを有効化 |

---

## コード実行フロー

### エントリポイント（SaveGame 生成 〜 保存 〜 ロード）

```
(セーブ - 同期)
UGameplayStatics::SaveGameToSlot(SaveGame, SlotName, UserIndex)   [GameplayStatics.cpp]
  ├─ UGameplayStatics::SaveGameToMemory(SaveGame, OutBytes)
  │    ├─ FMemoryWriter MemWriter(OutBytes)
  │    ├─ FSaveGameHeader Header(SaveGame->GetClass())
  │    │    └─ Header.Write(MemWriter)                              ← クラス名・バージョン
  │    └─ FObjectAndNameAsStringProxyArchive ProxyAr(MemWriter, true)
  │         ├─ ProxyAr.ArIsSaveGame = true                          ← CPF_SaveGame のみ書く
  │         └─ SaveGame->Serialize(ProxyAr)
  │              └─ Class->SerializeBin で UPROPERTY(SaveGame) のみ I/O
  │                   └─ UObject* は GetPathName() の文字列に変換
  └─ ISaveGameSystem::SaveGame(SlotName, UserIndex, Bytes)         [SaveGameSystem.h]
       └─ FGenericSaveGameSystem::SaveGame()                       [GenericPlatformSurvey.cpp]
            └─ ファイル書き込み: %LOCALAPPDATA%/.../SaveGames/SlotName.sav

(セーブ - 非同期、UE 5.1+)
UGameplayStatics::AsyncSaveGameToSlot(SaveGame, SlotName, UI, Delegate)
  └─ FAsyncTask<FAsyncSaveGameToSlotTask>->StartBackgroundTask()    ← TaskGraph
       ├─ ThreadPool で SaveGameToMemory() 実行                     ← データ準備をワーカー化
       └─ GameThread に戻って Delegate 発火

(ロード)
UGameplayStatics::LoadGameFromSlot(SlotName, UserIndex)
  ├─ ISaveGameSystem::LoadGame(SlotName, UserIndex, OutBytes)      ← ファイル読込
  └─ UGameplayStatics::LoadGameFromMemory(Bytes)
       ├─ FMemoryReader MemReader(Bytes)
       ├─ FSaveGameHeader Header
       │    └─ Header.Read(MemReader)                               ← クラス名取得
       ├─ UClass* SaveClass = StaticLoadClass(Header.SaveGameClassName)
       ├─ NewObject<USaveGame>(GetTransientPackage(), SaveClass)    ← インスタンス生成
       └─ FObjectAndNameAsStringProxyArchive ProxyAr(MemReader)
            ├─ ProxyAr.ArIsSaveGame = true
            └─ SaveGame->Serialize(ProxyAr)                         ← UPROPERTY(SaveGame) 復元

(スロット管理)
DoesSaveGameExist(SlotName, UI)  → ISaveGameSystem::DoesSaveGameExist()
DeleteGameInSlot(SlotName, UI)   → ISaveGameSystem::DeleteGame()
```

### フロー詳細

1. **SaveGame オブジェクト生成** — `CreateSaveGameObject<T>()` で `NewObject` ベースの USaveGame 派生インスタンスを生成。`Outer` は `GetTransientPackage()`（[[UObject/Details/c_outer_chain]]）。
2. **SaveGameToMemory** — `FMemoryWriter` を `FObjectAndNameAsStringProxyArchive` でラップ。`ArIsSaveGame = true` を立てることで `CPF_SaveGame` フラグのプロパティのみが処理対象になる（[[a_farchive]]）。
3. **UObject 参照の文字列化** — `FObjectAndNameAsStringProxyArchive` は UObject ポインタを `GetPathName()` 文字列として保存。ロード時に `StaticFindObject` でパスから復元するため、外部参照が壊れない。
4. **プラットフォーム抽象** — `ISaveGameSystem` がプラットフォーム差を吸収。Windows/Mac は `FGenericSaveGameSystem` がファイル書き込み、Console は専用ストレージ API。
5. **非同期版** — `AsyncSaveGameToSlot` は `FAsyncTask` でデータ準備（Serialize）をバックグラウンドに逃がし、完了時に GameThread でデリゲートを発火（[[AsyncTasks/Details/b_async_patterns]]）。
6. **ロード経路** — `FSaveGameHeader` から保存時のクラス名を読み、`StaticLoadClass` で UClass を解決して `NewObject` でインスタンス生成。その後 ProxyArchive で逆方向に Serialize して値を復元。
7. **バージョン互換** — SaveGame は `FCustomVersion` か独自のバージョンフィールドで管理。`Serialize()` 内で `Ar.IsLoading() && SaveVersion < N` 分岐により旧データの修正が可能。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `UGameplayStatics::SaveGameToSlot` / `LoadGameFromSlot` | `GameplayStatics.cpp` | 高レベル API |
| `UGameplayStatics::SaveGameToMemory` / `LoadGameFromMemory` | `GameplayStatics.cpp` | バイト配列 I/O |
| `UGameplayStatics::CreateSaveGameObject` | `GameplayStatics.cpp` | USaveGame 生成 |
| `FObjectAndNameAsStringProxyArchive` | `ArchiveObjectAndNameAsStringProxyArchive.h` | UObject* を文字列化するプロキシ |
| `FSaveGameHeader::Write` / `Read` | `GameplayStatics.cpp` | ヘッダ I/O |
| `ISaveGameSystem` | `SaveGameSystem.h` | プラットフォーム抽象インターフェース |
| `FGenericSaveGameSystem::SaveGame` / `LoadGame` | プラットフォーム別実装 | ファイル I/O |
| `FAsyncTask<FAsyncSaveGameToSlotTask>` | `GameplayStatics.cpp` | 非同期保存タスク |

---

## 関連ドキュメント

- [[a_farchive]] — `FMemoryWriter`/`FMemoryReader` の基底 `FArchive`
- [[b_asset_serialization]] — アセット（`.uasset`）のシリアライズ
- [[Reference/ref_serialization_api]] — `USaveGame` / `UGameplayStatics` のAPI
