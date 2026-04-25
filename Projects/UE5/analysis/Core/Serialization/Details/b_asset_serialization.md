# アセットシリアライゼーション（パッケージロード/セーブ）

- 上位: [[Serialization/01_overview]]
- 関連: [[a_farchive]] | [[c_save_game]]
- ソース: `CoreUObject/Public/UObject/LinkerLoad.h`, `CoreUObject/Public/UObject/LinkerSave.h`, `CoreUObject/Public/UObject/PackageFileSummary.h`, `CoreUObject/Public/Serialization/AsyncLoading2.h`

---

## 概要

UE5 のアセット（`.uasset`/`.umap`）は **`UPackage`** 単位でディスクに保存される。ロード時は `FLinkerLoad` が、セーブ時は `FLinkerSave` が `FArchive` 派生として動作し、パッケージヘッダ（`FPackageFileSummary`）・名前テーブル・インポート/エクスポートテーブルを処理する。UE5 では非同期ロードを担う **Zen Loader**（`FAsyncLoadingThread2`）への移行が進んでいる。

---

## .uasset のバイナリ構造

```
┌─────────────────────────────────┐
│ FPackageFileSummary             │ ← マジック(0x9E2A83C1)・バージョン・各テーブルのオフセット
├─────────────────────────────────┤
│ Name Table                      │ ← パッケージ内で使われる FName の文字列プール
├─────────────────────────────────┤
│ Import Table (FObjectImport[])  │ ← 他パッケージのオブジェクト参照（クラス等）
├─────────────────────────────────┤
│ Export Table (FObjectExport[])  │ ← このパッケージに含まれる UObject の一覧
├─────────────────────────────────┤
│ Export Data                     │ ← 各 UObject::Serialize() が書いたバイナリ
├─────────────────────────────────┤
│ BulkData                        │ ← テクスチャ・メッシュ等の大容量データ（遅延ロード可）
└─────────────────────────────────┘
```

### FPackageFileSummary の主要フィールド

| フィールド | 説明 |
|-----------|------|
| `Tag` | マジックナンバー `PACKAGE_FILE_TAG` (`0x9E2A83C1`) |
| `FileVersionUE` | UE ファイルバージョン |
| `CustomVersionContainer` | カスタムバージョン群 |
| `TotalHeaderSize` | サマリー + テーブル群のバイト数 |
| `NameCount` / `NameOffset` | 名前テーブルのエントリ数とオフセット |
| `ImportCount` / `ImportOffset` | インポートテーブル |
| `ExportCount` / `ExportOffset` | エクスポートテーブル |
| `PackageFlags` | `PKG_ContainsMap` 等のフラグ |

---

## パッケージロードフロー（FLinkerLoad）

```
LoadPackageAsync("/Game/Foo/Bar") または StaticLoadObject()
  │
  └─ FLinkerLoad::CreateLinker()
       ├─ FPackageFileSummary のデシリアライズ（ヘッダ読み込み）
       ├─ Name テーブル展開 → FName プールへ登録
       ├─ Import テーブル展開 → FObjectImport[] 構築
       ├─ Export テーブル展開 → FObjectExport[] 構築
       │
       └─ Preload 開始
            ├─ 各 Export の UObject を StaticConstructObject_Internal で生成
            │    （クラスは Import テーブルから解決）
            ├─ 各 UObject の UObject::Serialize(FLinkerLoad&) 呼び出し
            │    → UPROPERTY の値を FLinkerLoad から読み込む
            └─ PostLoad() 呼び出し（全 Export のロード完了後）
```

### FObjectImport と FObjectExport

```cpp
struct FObjectImport
{
    FName  ClassPackage;     // クラスが属するパッケージ名
    FName  ClassName;        // クラス名
    int32  OuterIndex;       // Outer の Import/Export テーブルインデックス
    FName  ObjectName;       // オブジェクト名
};

struct FObjectExport
{
    int32  ClassIndex;       // クラスの Import/Export インデックス
    int32  OuterIndex;       // Outer の Import/Export インデックス
    FName  ObjectName;       // オブジェクト名
    int64  SerialOffset;     // Export Data のファイルオフセット
    int64  SerialSize;       // データのバイト数
    EObjectFlags ObjectFlags;
};
```

---

## パッケージセーブフロー（FLinkerSave / SavePackage）

```
UPackage::SavePackage() (Editor 専用)
  └─ UPackageSaveContext 構築
       ├─ Export 収集（パッケージ内のすべての UObject を列挙）
       ├─ Import 収集（Export が参照する外部オブジェクトを列挙）
       ├─ Name テーブル構築
       ├─ FPackageFileSummary 構築
       │
       └─ FLinkerSave に書き込み
            ├─ FPackageFileSummary 書き込み
            ├─ Name テーブル書き込み
            ├─ Import テーブル書き込み
            ├─ Export テーブル書き込み
            └─ 各 UObject の Serialize() 呼び出し → Export Data に書き込み
```

---

## 非同期ロード（Zen Loader / FAsyncLoadingThread2）

UE5 の非同期ローダ。EDL（Event Driven Loader）から置き換わりつつある:

```mermaid
graph TD
    GT[GameThread: LoadPackageAsync 呼び出し] --> Q[ロードキューに追加]
    Q --> ALT[AsyncLoadingThread: I/O 実行<br>FPackage 生成・依存解決]
    ALT --> CR[CreateObjects<br>Export UObject 生成]
    CR --> SER[Serialize<br>プロパティ読み込み]
    SER --> PL[PostLoad<br>※ GameThread に同期]
    PL --> CB[完了コールバック<br>GameThread]
```

**GameThread との同期点** は `PostLoad()` のみ。それ以外はバックグラウンドで実行される。

### 非同期ロードの API

```cpp
// コールバック方式
FStreamableManager& SM = UAssetManager::GetStreamableManager();
SM.RequestAsyncLoad(SoftRef.ToSoftObjectPath(), FStreamableDelegate::CreateUObject(
    this, &AMyActor::OnAssetLoaded));

// 低レベル API
LoadPackageAsync(TEXT("/Game/Foo/Bar"),
    FLoadPackageAsyncDelegate::CreateLambda([](const FName& PackageName,
        UPackage* LoadedPackage, EAsyncLoadingResult::Type Result) {
        // 完了時コールバック
    }));

// 同期ロード（非推奨：ゲームスレッドをブロック）
UPackage* Pkg = LoadPackage(nullptr, TEXT("/Game/Foo/Bar"), LOAD_None);
```

---

## BulkData（大容量データの遅延ロード）

テクスチャ・スタティックメッシュのレンダリングデータ等は `FBulkData` で管理:

```cpp
class FBulkData
{
    // 必要なときだけロードする
    void* GetData();           // 同期ロード（ブロッキング）
    bool IsAvailableForUse();  // ロード済みか
    void RemoveBulkData();     // メモリ解放（再ロード可能）

    int64 GetBulkDataOffsetInFile();  // ファイル内オフセット
    int64 GetBulkDataSize();          // バイト数
};
```

`FByteBulkData` が最も一般的。`BULKDATA_PayloadAtEndOfFile` フラグで Export Data ではなく末尾（ストリーミング用）に配置される。

---

## SoftObjectPath / FSoftObjectPtr

ロードせずにアセットへの参照を保持するポインタ:

```cpp
UPROPERTY(EditAnywhere)
TSoftObjectPtr<UStaticMesh> MeshRef;

// 非同期ロード
if (!MeshRef.IsValid())
{
    UAssetManager::GetStreamableManager().RequestAsyncLoad(
        MeshRef.ToSoftObjectPath(), ...);
}
UStaticMesh* Mesh = MeshRef.Get();
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `s.AsyncLoadingThreadEnabled` | `1` | 非同期ロードスレッド有効化 |
| `s.UseBackgroundLevelStreaming` | `1` | レベルをバックグラウンドでロード |
| `s.MaxPackagesNotConsideredForShrinking` | `0` | Linker Pool サイズ管理 |

---

## コード実行フロー

### エントリポイント（パッケージロード 〜 セーブ 〜 BulkData）

```
(非同期ロード - Zen Loader)
LoadPackageAsync(PackagePath, CompletionDelegate)                  [PackageName.cpp]
  └─ FAsyncLoadingThread2::QueuePackage()                          [AsyncLoading2.cpp]
       ├─ PendingPackages に追加（GameThread から即帰る）
       └─ AsyncLoadingThread が拾う:
            ├─ FAsyncPackage2::ProcessPackageSummary()              ← FPackageFileSummary 読込
            │    └─ FPackageFileSummary::Serialize(LinkerArchive)
            ├─ ProcessPackageImportsAndExports()
            │    ├─ Name テーブル展開（FNamePool に登録）
            │    ├─ FObjectImport[] 構築
            │    └─ FObjectExport[] 構築
            ├─ ProcessExportBundleEntry() (各 Export ごと):
            │    ├─ StaticConstructObject_Internal()                ← UObject 生成
            │    └─ Export->Object->Serialize(FLinkerLoad&)
            │         └─ Class->SerializeBin → FProperty::SerializeBinProperty
            └─ GameThread 同期点: FlushAsyncLoading()                ← PostLoad 発火
                 └─ Object->PostLoad()                              ← ロード後フック

(同期ロード - レガシー)
LoadPackage(nullptr, PackagePath, LOAD_None)                       [Package.cpp]
  └─ FLinkerLoad::CreateLinker()                                   [LinkerLoad.cpp]
       └─ Preload(Object) で同期的に上記と同じシーケンス

(セーブ - エディタ専用)
UPackage::SavePackage(Package, Asset, Filename, SaveArgs)         [SavePackage.cpp]
  ├─ FSaveContext::Setup() で Export/Import 収集
  ├─ FLinkerSave::Save()                                           [LinkerSave.cpp]
  │    ├─ Name テーブル構築
  │    ├─ FPackageFileSummary 書き込み（オフセットは仮）
  │    ├─ Name/Import/Export テーブル書き込み
  │    ├─ for each Export: Object->Serialize(FLinkerSave&)
  │    │    └─ Class->SerializeBin → 各 FProperty を書く
  │    └─ FPackageFileSummary を実オフセットで書き直す              ← シーク戻る
  └─ FBulkData の遅延書き込み
       └─ BULKDATA_PayloadAtEndOfFile フラグなら .uasset 末尾 or .ubulk 別ファイル

(BulkData ロード)
FByteBulkData::GetData()                                          [BulkData.cpp]
  ├─ if (Loaded) return Pointer
  └─ LoadBulkDataWithFileReader()
       └─ FArchive Reader で BulkDataOffsetInFile から SerialSize を読む
```

### フロー詳細

1. **非同期ロード起動** — `LoadPackageAsync` が `FAsyncLoadingThread2::QueuePackage` でキュー登録。GameThread はブロックされず、AsyncLoadingThread がバックグラウンドで処理。
2. **Summary/テーブル展開** — `FAsyncPackage2` が `FPackageFileSummary` を先頭から読み、Name/Import/Export テーブルを順次デシリアライズ。Name は `FNamePool` に登録（[[Containers/Details/b_string_types]]）。
3. **Export 構築** — 各 `FObjectExport` に対して `StaticConstructObject_Internal` で UObject を生成。クラスは `Import` テーブルから解決される（[[UObject/Details/a_lifecycle]]）。
4. **Serialize 復元** — 生成済み UObject の `Serialize(FLinkerLoad&)` を呼ぶ。`UStruct::SerializeBin` がリフレクション経由で UPROPERTY を順次復元。
5. **GameThread 同期** — 全 Export の Serialize 完了後、`FlushAsyncLoading` 経由で GameThread に戻り、`PostLoad()` が呼ばれる（async loading の唯一の同期点）。
6. **セーブ経路** — `UPackage::SavePackage` が Export/Import を収集し、`FLinkerSave` で Summary/テーブル/Export Data を順次書く。Summary は実オフセット確定後にシーク戻して上書き。
7. **BulkData** — メッシュ・テクスチャ等の大容量データは `FBulkData` が遅延ロード。`GetData()` 時に `BulkDataOffsetInFile` から必要分だけ読み込む。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `LoadPackageAsync` | `PackageName.cpp` | 非同期ロード API |
| `FAsyncLoadingThread2::QueuePackage` | `AsyncLoading2.cpp` | Zen Loader キュー |
| `FAsyncPackage2::ProcessPackageSummary` | `AsyncLoading2.cpp` | Summary/テーブル展開 |
| `FLinkerLoad::CreateLinker` / `Preload` | `LinkerLoad.cpp` | パッケージ → UObject 復元 |
| `FPackageFileSummary::Serialize` | `PackageFileSummary.cpp` | ヘッダ I/O |
| `UPackage::SavePackage` | `SavePackage.cpp` | エディタセーブドライバ |
| `FLinkerSave::Save` | `LinkerSave.cpp` | パッケージ書き込み本体 |
| `FByteBulkData::GetData` | `BulkData.cpp` | BulkData 遅延ロード |
| `FlushAsyncLoading` | `AsyncLoading2.cpp` | GameThread 同期点 |

---

## 関連ドキュメント

- [[a_farchive]] — `FLinkerLoad`/`FLinkerSave` の基底 `FArchive`
- [[c_save_game]] — より高レベルな SaveGame フロー
- [[Reference/ref_serialization_api]] — `FLinkerLoad` / `LoadPackageAsync` の API
