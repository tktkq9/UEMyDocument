# UObject / UObjectBaseUtility API リファレンス

- 上位: [[UObject/01_overview]]
- 関連: [[Details/a_lifecycle]] | [[Details/b_garbage_collection]] | [[Details/c_outer_chain]]
- ソース: `CoreUObject/Public/UObject/Object.h`, `CoreUObject/Public/UObject/UObjectBaseUtility.h`

---

## オブジェクト生成

```cpp
// NewObject<T> — 最も一般的な生成関数
T* NewObject<T>(
    UObject* Outer,           // 所有者（null = GetTransientPackage()）
    UClass*  Class,           // クラス（null = T::StaticClass()）
    FName    Name,            // オブジェクト名（NAME_None = 自動生成）
    EObjectFlags Flags,       // RF_* フラグ
    UObject* Template         // テンプレートオブジェクト（null = CDO）
)

// 例
UMyComponent* Comp = NewObject<UMyComponent>(this, TEXT("MyComp"));
UMyObject*    Obj  = NewObject<UMyObject>(GetTransientPackage(), UMyObject::StaticClass(), NAME_None, RF_Transient);

// CreateDefaultSubobject — コンストラクタ内でのみ使用可
UMyComponent* Comp = CreateDefaultSubobject<UMyComponent>(TEXT("Comp"));

// DuplicateObject — 既存オブジェクトのディープコピー
T* DuplicateObject<T>(T* SourceObject, UObject* Outer, FName Name = NAME_None)
UMyObject* Copy = DuplicateObject<UMyObject>(Original, GetTransientPackage());

// StaticConstructObject_Internal — 低レベル生成（通常は使わない）
```

---

## キャスト

```cpp
// Cast<T> — 失敗時 nullptr を返す（UObject 間の安全なダウンキャスト）
AMyActor* Actor = Cast<AMyActor>(SomeObject);

// CastChecked<T> — 失敗時アサート（デバッグ用）
AMyActor* Actor = CastChecked<AMyActor>(SomeObject);

// IsA<T> — 型チェック（継承を考慮）
bool bIsActor = Object->IsA<AActor>();
bool bIsActor2 = Object->IsA(AActor::StaticClass());
```

---

## 検索・ロード

```cpp
// StaticFindObject — ロード済みオブジェクトを名前で検索
UObject* Found = StaticFindObject(UClass*, UObject* Outer, const TCHAR* Name, bool bExactClass=false);
AMyActor* Actor = Cast<AMyActor>(
    StaticFindObject(AMyActor::StaticClass(), GetWorld(), TEXT("MyActor")));

// StaticLoadObject — ディスクからロード（同期）
UObject* Loaded = StaticLoadObject(UClass*, UObject* Outer, const TCHAR* Name);
UTexture2D* Tex = Cast<UTexture2D>(
    StaticLoadObject(UTexture2D::StaticClass(), nullptr, TEXT("/Game/Textures/T_MyTex")));

// LoadObject<T> — ラッパー（型付き版）
UTexture2D* Tex = LoadObject<UTexture2D>(nullptr, TEXT("/Game/Textures/T_MyTex"));

// FindObject<T>
UObject* Obj = FindObject<UObject>(Package, TEXT("ObjectName"));
```

---

## UObject メソッド

### 識別・名前

```cpp
// オブジェクト名
FString  GetName()     const;   // "MyActor_0"
FName    GetFName()    const;   // FName 版
FString  GetFullName() const;   // "/Game/Map.Map:PersistentLevel.MyActor_0"
FString  GetPathName() const;   // フルパス文字列
FString  GetPathName(UObject* StopOuter) const;  // StopOuter を起点にした相対パス

// クラス
UClass*  GetClass() const;
bool     IsA(UClass*) const;

// 外部オブジェクト
UObject* GetOuter() const;
UPackage* GetOutermost() const;  // 最上位の UPackage
bool     IsIn(UObject* SomeOuter) const;

// 有効性確認
bool     IsValid() const;
bool     IsPendingKill() const;   // UE5 では IsValid で代替
bool     IsUnreachable() const;   // GC マークされているか
bool     HasAnyFlags(EObjectFlags) const;
bool     HasAllFlags(EObjectFlags) const;
```

### ライフサイクルフック（オーバーライド可能）

```cpp
virtual void PostInitProperties();          // NewObject 後の初期化
virtual void PostLoad();                    // デシリアライズ完了後
virtual void PreSave(FObjectPreSaveContext);// 保存前
virtual void PostSaveRoot(FObjectPostSaveRootContext);
virtual void BeginDestroy();                // 破棄開始（GC から呼ばれる）
virtual void FinishDestroy();               // 破棄完了
virtual bool IsReadyForFinishDestroy();     // BeginDestroy 後に GC がポーリング
virtual void Serialize(FArchive& Ar);       // シリアライズ
```

### オブジェクトフラグ操作

```cpp
void      SetFlags(EObjectFlags Flags);
void      ClearFlags(EObjectFlags Flags);
EObjectFlags GetFlags() const;
bool      HasAnyFlags(EObjectFlags) const;

// 主要フラグ
RF_Public          // パッケージ外から参照可能
RF_Standalone      // GC ルートセットに含める
RF_Transient       // 保存しない
RF_ArchetypeObject // CDO / Archetype であることを示す
RF_ClassDefaultObject  // CDO であることを示す
```

---

## UObjectBaseUtility メソッド

```cpp
// 参照されているか確認
bool IsReferenced(UObject* Res, bool bCheckSubObjects=false);

// ソフト参照の解決
FSoftObjectPath GetSoftObjectPath() const;

// マーク（GC 内部で使われる）
void Mark(EInternalObjectFlags Flags);
bool HasAnyInternalFlags(EInternalObjectFlags) const;

// 名前の変更
void Rename(const TCHAR* NewName, UObject* NewOuter, ERenameFlags Flags);
```

---

## グローバルユーティリティ

```cpp
// 有効性チェック（最も安全な方法）
bool IsValid(const UObject* Test);

// ガベージコレクション
void CollectGarbage(EObjectFlags KeepFlags, bool bPerformFullPurge=false);
void GarbageCollectSharedClusters();

// GC 中かどうか
bool IsGarbageCollecting();

// 全 UObject イテレート
for (TObjectIterator<UMyClass> It; It; ++It)
{
    UMyClass* Obj = *It;
}

// 特定クラスの全インスタンス取得
TArray<UObject*> Objects;
GetObjectsOfClass(UMyClass::StaticClass(), Objects);

// 全参照オブジェクト取得
TArray<UObject*> Referenced;
FReferenceFinder Finder(Referenced, nullptr, false, true);
Finder.FindReferences(RootObject);
```

---

## EObjectFlags 主要値

| フラグ | 説明 |
|--------|------|
| `RF_Public` | パッケージ外から参照可能 |
| `RF_Standalone` | 明示的に参照されなくても保持 |
| `RF_MarkAsNative` | ネイティブ C++ オブジェクト |
| `RF_Transactional` | Undo/Redo 対象（Editor） |
| `RF_ClassDefaultObject` | CDO であることを示す |
| `RF_ArchetypeObject` | Archetype（テンプレート） |
| `RF_Transient` | ディスクに保存しない |
| `RF_NeedLoad` | ロード待ちオブジェクト |
| `RF_NeedPostLoad` | PostLoad 待ち |
| `RF_BeginDestroyed` | BeginDestroy 呼び済み |
| `RF_FinishDestroyed` | FinishDestroy 呼び済み |

---

## 関連ドキュメント

- [[Details/a_lifecycle]] — NewObject フロー・ライフサイクルフックの詳細
- [[Details/b_garbage_collection]] — GC・FGCObject
- [[Details/c_outer_chain]] — Outer チェーン・パス解決
- [[ref_gc_api]] — GC 関連 API
