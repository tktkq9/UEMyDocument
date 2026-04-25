# Core ドキュメント チェックリスト

## 概要
- [x] `01_core_overview.md` … Core 全体概要（2026-04-24 完了）
- [x] `_source_map.md` … ソースマップ（2026-04-24 完了）

---

## サブフォルダ構成

### UObject — オブジェクトシステム
- [x] `UObject/01_overview.md` … UObject 概要（2026-04-24 完了）
- [x] `UObject/Details/a_lifecycle.md` … NewObject・CDO・PostInitProperties・ConditionalBeginDestroy（2026-04-24）
- [x] `UObject/Details/b_garbage_collection.md` … FGCObject・AddReferencedObjects・Mark-Sweep・クラスタリング（2026-04-24）
- [x] `UObject/Details/c_outer_chain.md` … Outer チェーン・GetTransientPackage・Rename・パス解決（2026-04-24）
- [x] `UObject/Details/d_class_default_object.md` … CDO 生成・GetDefaultObject・Archetype チェーン（2026-04-24）
- [x] `UObject/Reference/ref_uobject_api.md` … UObject / UObjectBaseUtility API
- [x] `UObject/Reference/ref_gc_api.md` … GC 関連関数・UPROPERTY GC 指定子

### Reflection — リフレクションシステム
- [x] `Reflection/01_overview.md` … Reflection 概要（2026-04-24 完了）
- [x] `Reflection/Details/a_uclass.md` … UClass・ClassFlags・Interfaces・StaticClass()（2026-04-24）
- [x] `Reflection/Details/b_fproperty.md` … FProperty 階層・FNumericProperty/FObjectProperty/FStructProperty（2026-04-24）
- [x] `Reflection/Details/c_ufunction.md` … UFunction・FunctionFlags・NativeFunction・ProcessEvent（2026-04-24）
- [x] `Reflection/Details/d_metadata.md` … UMETA・メタデータスペシファイア・EditCondition・ToolTip（2026-04-24）
- [x] `Reflection/Reference/ref_reflection_api.md` … UClass / FProperty / UFunction API
- [x] `Reflection/Reference/ref_macros.md` … UCLASS/UPROPERTY/UFUNCTION/USTRUCT マクロ全一覧

### Serialization — シリアライゼーション
- [x] `Serialization/01_overview.md` … Serialization 概要（2026-04-24 完了）
- [x] `Serialization/Details/a_farchive.md` … FArchive・operator<<・IsLoading/IsSaving・カスタムバージョン（2026-04-25）
- [x] `Serialization/Details/b_asset_serialization.md` … アセットレジストリ・FLinkerLoad/Save・パッケージ（2026-04-25）
- [x] `Serialization/Details/c_save_game.md` … USaveGame・UGameplayStatics::SaveGameToSlot・バイナリ/JSON（2026-04-25）
- [x] `Serialization/Reference/ref_serialization_api.md` … FArchive / FStructuredArchive API

### AsyncTasks — 非同期処理
- [x] `AsyncTasks/01_overview.md` … AsyncTasks 概要（2026-04-24 完了）
- [x] `AsyncTasks/Details/a_task_graph.md` … FTaskGraphInterface・FBaseGraphTask・前提条件・NamedThread
- [x] `AsyncTasks/Details/b_async_patterns.md` … AsyncTask・Async()・FRunnable・FQueuedThreadPool
- [x] `AsyncTasks/Details/c_game_thread.md` … GameThread/RenderThread/RHIThread 連携・ENQUEUE_RENDER_COMMAND
- [x] `AsyncTasks/Details/d_parallel_for.md` … ParallelFor・EParallelForFlags・バッチサイズ
- [x] `AsyncTasks/Reference/ref_async_api.md` … Async / FAsyncTask / FRunnable API

### Delegates — デリゲート・イベント
- [x] `Delegates/01_overview.md` … Delegates 概要（2026-04-24 完了）
- [x] `Delegates/Details/a_delegate_types.md` … TDelegate・TMulticastDelegate・DECLARE_DELEGATE マクロ群
- [x] `Delegates/Details/b_dynamic_delegates.md` … FDynamicDelegate・DECLARE_DYNAMIC_MULTICAST_DELEGATE・BP 対応
- [x] `Delegates/Details/c_timer.md` … FTimerManager・SetTimer・SetTimerForNextTick・ClearTimer
- [x] `Delegates/Reference/ref_delegate_api.md` … TDelegate / TMulticastDelegate / FTimerManager API

### Containers — コンテナ・文字列
- [x] `Containers/01_overview.md` … Containers 概要（2026-04-24 完了）
- [x] `Containers/Details/a_array_map_set.md` … TArray・TMap・TSet・TSparseArray・メモリレイアウト
- [x] `Containers/Details/b_string_types.md` … FString・FName・FText・変換・ローカライズ
- [x] `Containers/Details/c_smart_pointers.md` … TSharedPtr・TWeakPtr・TUniquePtr・TSharedRef
- [x] `Containers/Reference/ref_containers_api.md` … TArray / TMap / TSet 主要メソッド一覧

---

## 進捗サマリ

| サブフォルダ | 概要 | Details | Reference | 完了 |
|------------|------|---------|-----------|------|
| UObject | 1/1 | 4/4 | 2/2 | 7/7 |
| Reflection | 1/1 | 4/4 | 2/2 | 7/7 |
| Serialization | 1/1 | 3/3 | 1/1 | 5/5 |
| AsyncTasks | 1/1 | 4/4 | 1/1 | 6/6 |
| Delegates | 1/1 | 3/3 | 1/1 | 5/5 |
| Containers | 1/1 | 3/3 | 1/1 | 5/5 |

**合計**: 概要 1 + ソースマップ 1 + サブフォルダ概要 6 + Details 21 + Reference 8 = **37 ファイル**

---

## Ph4: コード実行フロー追記（2026-04-25 完了）

全 28 ファイル（overview 7 本 + Details 21 本）に `## コード実行フロー` セクションを追記。
ASCII コールツリー → 番号付きフロー詳細（wikilink 付き）→ 関与クラス・関数一覧（3 列表）の形式。

- [x] `01_core_overview.md`
- [x] `UObject/01_overview.md` + Details 4 本
- [x] `Reflection/01_overview.md` + Details 4 本
- [x] `Serialization/01_overview.md` + Details 3 本
- [x] `AsyncTasks/01_overview.md` + Details 4 本
- [x] `Delegates/01_overview.md` + Details 3 本
- [x] `Containers/01_overview.md` + Details 3 本
