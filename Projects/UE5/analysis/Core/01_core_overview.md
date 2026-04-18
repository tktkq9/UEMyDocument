# Core システム全体概要

- 取得対象: `Engine/Source/Runtime/CoreUObject/`, `Engine/Source/Runtime/Core/`
- 上位: [[UE5 解析インデックス]]

---

## UE5 Core システムの構成

エンジン全体の基盤となるオブジェクトシステム・リフレクション・GC・非同期処理。

| 機能 | クラス / 仕組み | 説明 |
|------|-------------|------|
| UObject | `UObject` | 全オブジェクトの基底・GC 対象 |
| リフレクション | `UCLASS / UPROPERTY / UFUNCTION` | 型情報・Blueprint 公開 |
| GC | `FGarbageCollector` | マーク・スイープ GC |
| CDO | Class Default Object | クラスのデフォルト値保持 |
| シリアライズ | `FArchive` | セーブ・ロード・ネット複製 |
| アセット | `UAssetManager` / `FStreamableManager` | 非同期ロード |
| デリゲート | `TDelegate / TMulticastDelegate` | 型安全なコールバック |
| TaskGraph | `FTaskGraphInterface` | 並列タスク・スレッドプール |
| Timer | `FTimerManager` | タイマー管理 |

---

## UObject ライフサイクル

```
NewObject<T>() / CreateDefaultSubobject<T>()
  └─ UClass::CreateDefaultObject()  // CDO 作成
      └─ UObject::PostInitProperties()
          └─ UObject::PostLoad()     // ロード時
              └─ GC マーク対象に追加

[GC 時]
FGarbageCollector::CollectGarbage()
  ├─ Mark: ReachabilityAnalysis（参照グラフ巡回）
  └─ Sweep: ConditionalBeginDestroy() → BeginDestroy() → FinishDestroy()
```

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `UObjectBase.h/.cpp` | UObject 基底 |
| `GarbageCollection.h/.cpp` | GC 実装 |
| `Class.h/.cpp` | UClass・UCLASS マクロ展開 |
| `Archive.h/.cpp` | シリアライズ基底 |
| `AssetManager.h/.cpp` | アセット管理 |
| `TaskGraphInterfaces.h/.cpp` | タスクグラフ |
| `DelegateSignatureImpl.inl` | デリゲート実装 |
| `HAL/Platform.h` | プラットフォーム抽象化 |

---

## サブシステムドキュメント

| ドキュメント | 内容 |
|------------|------|
| `Details/a_uobject_lifecycle.md` | NewObject・CDO・GC・BeginDestroy |
| `Details/b_reflection.md` | UCLASS/UPROPERTY/UFUNCTION マクロ展開・型情報 |
| `Details/c_serialization.md` | FArchive・SaveGame・ネット複製との関係 |
| `Details/d_async_tasks.md` | TaskGraph・Async・FRunnable・スレッドモデル |
| `Details/e_delegates.md` | TDelegate・TMulticastDelegate・FTimerHandle |
