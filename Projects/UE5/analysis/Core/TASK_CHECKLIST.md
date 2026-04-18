# Core ドキュメント チェックリスト

## 概要
- [ ] `01_core_overview.md` … Core 全体概要（更新予定）
- [ ] `_source_map.md` … ソースマップ

---

## サブフォルダ構成

### UObject — オブジェクトシステム
- [ ] `UObject/01_overview.md` … UObject 概要
- [ ] `UObject/Details/a_lifecycle.md` … NewObject・CDO・PostInitProperties・ConditionalBeginDestroy
- [ ] `UObject/Details/b_garbage_collection.md` … FGCObject・AddReferencedObjects・Mark-Sweep・クラスタリング
- [ ] `UObject/Details/c_outer_chain.md` … Outer チェーン・GetTransientPackage・Rename・パス解決
- [ ] `UObject/Details/d_class_default_object.md` … CDO 生成・GetDefaultObject・Archetype チェーン
- [ ] `UObject/Reference/ref_uobject_api.md` … UObject / UObjectBaseUtility API
- [ ] `UObject/Reference/ref_gc_api.md` … GC 関連関数・UPROPERTY GC 指定子

### Reflection — リフレクションシステム
- [ ] `Reflection/01_overview.md` … Reflection 概要
- [ ] `Reflection/Details/a_uclass.md` … UClass・ClassFlags・Interfaces・StaticClass()
- [ ] `Reflection/Details/b_fproperty.md` … FProperty 階層・FNumericProperty/FObjectProperty/FStructProperty
- [ ] `Reflection/Details/c_ufunction.md` … UFunction・FunctionFlags・NativeFunction・ProcessEvent
- [ ] `Reflection/Details/d_metadata.md` … UMETA・メタデータスペシファイア・EditCondition・ToolTip
- [ ] `Reflection/Reference/ref_reflection_api.md` … UClass / FProperty / UFunction API
- [ ] `Reflection/Reference/ref_macros.md` … UCLASS/UPROPERTY/UFUNCTION/USTRUCT マクロ全一覧

### Serialization — シリアライゼーション
- [ ] `Serialization/01_overview.md` … Serialization 概要
- [ ] `Serialization/Details/a_farchive.md` … FArchive・operator<<・IsLoading/IsSaving・カスタムバージョン
- [ ] `Serialization/Details/b_asset_serialization.md` … アセットレジストリ・FLinkerLoad/Save・パッケージ
- [ ] `Serialization/Details/c_save_game.md` … USaveGame・UGameplayStatics::SaveGameToSlot・バイナリ/JSON
- [ ] `Serialization/Reference/ref_serialization_api.md` … FArchive / FStructuredArchive API

### AsyncTasks — 非同期処理
- [ ] `AsyncTasks/01_overview.md` … AsyncTasks 概要
- [ ] `AsyncTasks/Details/a_task_graph.md` … FTaskGraphInterface・FBaseGraphTask・前提条件・NamedThread
- [ ] `AsyncTasks/Details/b_async_patterns.md` … AsyncTask・Async()・FRunnable・FQueuedThreadPool
- [ ] `AsyncTasks/Details/c_game_thread.md` … GameThread/RenderThread/RHIThread 連携・ENQUEUE_RENDER_COMMAND
- [ ] `AsyncTasks/Details/d_parallel_for.md` … ParallelFor・EParallelForFlags・バッチサイズ
- [ ] `AsyncTasks/Reference/ref_async_api.md` … Async / FAsyncTask / FRunnable API

### Delegates — デリゲート・イベント
- [ ] `Delegates/01_overview.md` … Delegates 概要
- [ ] `Delegates/Details/a_delegate_types.md` … TDelegate・TMulticastDelegate・DECLARE_DELEGATE マクロ群
- [ ] `Delegates/Details/b_dynamic_delegates.md` … FDynamicDelegate・DECLARE_DYNAMIC_MULTICAST_DELEGATE・BP 対応
- [ ] `Delegates/Details/c_timer.md` … FTimerManager・SetTimer・SetTimerForNextTick・ClearTimer
- [ ] `Delegates/Reference/ref_delegate_api.md` … TDelegate / TMulticastDelegate / FTimerManager API

### Containers — コンテナ・文字列
- [ ] `Containers/01_overview.md` … Containers 概要
- [ ] `Containers/Details/a_array_map_set.md` … TArray・TMap・TSet・TSparseArray・メモリレイアウト
- [ ] `Containers/Details/b_string_types.md` … FString・FName・FText・変換・ローカライズ
- [ ] `Containers/Details/c_smart_pointers.md` … TSharedPtr・TWeakPtr・TUniquePtr・TSharedRef
- [ ] `Containers/Reference/ref_containers_api.md` … TArray / TMap / TSet 主要メソッド一覧

---

## 進捗サマリ

| サブフォルダ | 概要 | Details | Reference | 完了 |
|------------|------|---------|-----------|------|
| UObject | 0/1 | 0/4 | 0/2 | 0/7 |
| Reflection | 0/1 | 0/4 | 0/2 | 0/7 |
| Serialization | 0/1 | 0/3 | 0/1 | 0/5 |
| AsyncTasks | 0/1 | 0/4 | 0/1 | 0/6 |
| Delegates | 0/1 | 0/3 | 0/1 | 0/5 |
| Containers | 0/1 | 0/3 | 0/1 | 0/5 |

**合計**: 概要 1 + ソースマップ 1 + サブフォルダ概要 6 + Details 21 + Reference 8 = **37 ファイル**
