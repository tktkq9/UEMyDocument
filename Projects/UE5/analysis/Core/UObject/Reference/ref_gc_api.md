# GC 関連 API リファレンス

- 上位: [[UObject/01_overview]]
- 関連: [[Details/b_garbage_collection]] | [[ref_uobject_api]]
- ソース: `CoreUObject/Public/UObject/GarbageCollection.h`, `CoreUObject/Public/UObject/GCObject.h`

---

## GC の手動実行

```cpp
#include "UObject/GarbageCollection.h"

// フル GC を実行（エンジンの定期 GC ではなく、即時実行）
// KeepFlags: このフラグを持つオブジェクトは回収しない
CollectGarbage(RF_NoFlags);               // すべて回収対象
CollectGarbage(RF_Standalone, true);      // bPerformFullPurge=true で完全パージ

// GC 中か確認
bool bCollecting = IsGarbageCollecting();

// GC を一時禁止（ネストは禁止）
FGCScopeGuard GCGuard;
// GCGuard のスコープ内では GC は実行されない
```

---

## UPROPERTY — GC への参照登録

```cpp
// UPROPERTY を付けることで GC が参照を追跡する（最も推奨）
UPROPERTY()
UMyObject* StrongRef;  // GC が生きている間このオブジェクトを保持

// 配列・マップも追跡される
UPROPERTY()
TArray<UMyObject*> ObjectList;

UPROPERTY()
TMap<FName, UMyObject*> ObjectMap;
```

`UPROPERTY()` のないポインタは GC に見えないため、GC 実行後にダングリングポインタになりうる。

---

## FGCObject — 非 UObject からの参照

```cpp
#include "UObject/GCObject.h"

class FMyManager : public FGCObject
{
public:
    // 参照を GC に伝える（必須）
    virtual void AddReferencedObjects(FReferenceCollector& Collector) override
    {
        Collector.AddReferencedObject(ManagedObject);
        Collector.AddReferencedObjects(ObjectList);
    }

    virtual FString GetReferencerName() const override
    {
        return TEXT("FMyManager");
    }

private:
    UMyObject* ManagedObject = nullptr;
    TArray<UMyObject*> ObjectList;
};
```

`FGCObject` は Singleton や純粋 C++ クラスが UObject を保持するための公式手段。

---

## TStrongObjectPtr — ルート参照

GC ルートセットに直接オブジェクトを固定する:

```cpp
#include "UObject/StrongObjectPtr.h"

TStrongObjectPtr<UMyObject> StrongPtr(NewObject<UMyObject>());

// StrongPtr が生きている間、ガベージコレクションされない
// デストラクタで自動解除
UMyObject* Raw = StrongPtr.Get();
```

エディタや非 UObject コンテキストで一時的にオブジェクトを保持するのに使う。

---

## TWeakObjectPtr — 弱参照

GC に影響せず、対象が回収されたことを検知できる:

```cpp
#include "UObject/WeakObjectPtr.h"

TWeakObjectPtr<AMyActor> WeakActor = SomeActor;

// 有効チェック（GC 後は false になる）
if (WeakActor.IsValid())
{
    WeakActor->DoSomething();
}

// 生ポインタ取得（IsValid が false なら nullptr）
AMyActor* Raw = WeakActor.Get();

// 明示的なリセット
WeakActor.Reset();

// null チェック
bool bIsNull = WeakActor == nullptr;
```

---

## FReferenceCollector — AddReferencedObjects 内で使用

```cpp
virtual void AddReferencedObjects(FReferenceCollector& Collector) override
{
    // 単一オブジェクト
    Collector.AddReferencedObject(MyObject);

    // 配列
    Collector.AddReferencedObjects(MyArray);

    // マップ（値側を追跡）
    for (auto& Pair : MyMap)
    {
        Collector.AddReferencedObject(Pair.Value);
    }

    // 条件付き追跡
    if (bShouldTrack)
    {
        Collector.AddReferencedObject(ConditionalObject);
    }
}
```

---

## GC クラスタリング

```cpp
// クラスター作成（UObject::CreateCluster 相当の設定）
// UCLASS(Within=...) または SetFlags(RF_ClusterRoot) で制御

// クラスタリングを有効にするか確認
bool bClustered = Object->CanBeClusterRoot();

// クラスタに追加するオブジェクトを列挙
virtual void CreateCluster() override;

// クラスタ化されたオブジェクトに参照を追加
virtual void AddToCluster(UObjectBaseUtility* ClusterRootOrObjectFromCluster,
                          bool bAddAsMutableObject = false);
```

---

## GC 関連 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `gc.TimeBetweenPurgingPendingKillObjects` | `60.0` | GC 実行間隔（秒） |
| `gc.MaxObjectsNotConsideredByGC` | `0` | GC の対象外にする最大オブジェクト数 |
| `gc.FlushStreamingOnGC` | `0` | GC 時にストリーミングをフラッシュ |
| `gc.NumRetriesBeforeForcingGC` | `10` | 強制 GC までの待機回数 |
| `gc.IncrementalGC` | `1` | インクリメンタル GC の有効化 |
| `gc.IncrementalGCTimePerFrame` | `0.002` | フレームあたりの GC 処理時間（秒） |
| `gc.CreateGCClusters` | `1` | GC クラスタリングの有効化 |

---

## ルートセット管理

```cpp
// オブジェクトをルートセットに追加（GC から保護）
Object->AddToRoot();

// ルートセットから削除（GC の対象になる）
Object->RemoveFromRoot();

// ルートセットにあるか確認
bool bInRoot = Object->IsRooted();
```

`AddToRoot` は `TStrongObjectPtr` よりも粗い制御なので、スコープが明確な場合は `TStrongObjectPtr` を優先する。

---

## 参照グラフのデバッグ

```cpp
// 特定オブジェクトを参照しているオブジェクト一覧を取得
TArray<UObject*> Referencers;
FReferencerInformationList RefInfo;
IsReferenced(Object, RF_NoFlags, EInternalObjectFlags::None, true, &RefInfo);

// コンソールコマンド（実行時）
// obj refs class=MyClass name=MyObj — 参照元を表示
// obj refs class=MyClass name=MyObj — 参照先を表示
```

---

## 関連ドキュメント

- [[Details/b_garbage_collection]] — Mark&Sweep フロー・クラスタリングの概念
- [[ref_uobject_api]] — UObject の基本 API（NewObject・IsValid 等）
