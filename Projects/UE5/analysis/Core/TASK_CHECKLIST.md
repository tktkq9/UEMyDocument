# Core ドキュメント チェックリスト

## 概要
- [ ] `01_core_overview.md` … Core システム全体概要

---

## Details（詳細解析）

- [ ] `Details/a_uobject_lifecycle.md` … NewObject・CDO・PostInitProperties・GC フロー
- [ ] `Details/b_reflection.md` … UCLASS/UPROPERTY/UFUNCTION 展開・UField・FProperty
- [ ] `Details/c_serialization.md` … FArchive・SaveGame・アセットシリアライズ・バージョニング
- [ ] `Details/d_async_tasks.md` … FTaskGraphInterface・Async・FRunnable・GameThread/RenderThread
- [ ] `Details/e_delegates.md` … TDelegate・TMulticastDelegate・FTimerManager・WeakPtr 対応

## Reference（リファレンス）

- [ ] `Reference/ref_uobject_api.md` … UObject 主要 API・GC 関連関数
- [ ] `Reference/ref_macros.md` … UCLASS/UPROPERTY/UFUNCTION/USTRUCT マクロ一覧
- [ ] `Reference/ref_async_api.md` … AsyncTask・Async・FTimerHandle API

---

合計: 概要 1 + Details 5 + Reference 3 = **9 ファイル**
