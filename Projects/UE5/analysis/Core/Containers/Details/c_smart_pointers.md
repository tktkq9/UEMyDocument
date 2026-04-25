# スマートポインタ（TSharedPtr / TWeakPtr / TUniquePtr）

- 上位: [[Containers/01_overview]]
- 関連: [[a_array_map_set]] | [[b_string_types]]
- ソース: `Core/Public/Templates/SharedPointer.h`, `Core/Public/Templates/UniquePtr.h`

---

## 概要

UE5 のスマートポインタは **GC 管理外**（非 UObject）のオブジェクト管理に使う。`UObject` には使わない（GC と二重管理になる）。

| 型 | 用途 | 所有権 | スレッド安全 |
|----|------|--------|------------|
| `TSharedPtr<T>` | 共有所有（参照カウント） | 共有 | 選択可 |
| `TSharedRef<T>` | null なし共有所有 | 共有 | 選択可 |
| `TWeakPtr<T>` | 非所有・ダングリング検知 | なし | 選択可 |
| `TUniquePtr<T>` | 単独所有（軽量）| 排他 | - |

---

## TSharedPtr

`std::shared_ptr<T>` 相当。参照カウントが 0 になった時点で delete される。

```cpp
#include "Templates/SharedPointer.h"

// 生成（MakeShared が推奨: 参照カウントブロックとオブジェクトを一括確保）
TSharedPtr<FMyData> Ptr = MakeShared<FMyData>(ConstructorArg);

// 古い生成方法（2 回アロケーション）
TSharedPtr<FMyData> Ptr2 = MakeShareable(new FMyData());

// コピー（参照カウント増加）
TSharedPtr<FMyData> Ptr3 = Ptr;

// アクセス
if (Ptr.IsValid())
{
    Ptr->DoSomething();
    FMyData& Ref = *Ptr;
}

// 解放（参照カウント減少）
Ptr.Reset();

// 生ポインタ取得（ライフタイムに注意）
FMyData* Raw = Ptr.Get();

// 参照カウント確認
int32 RefCount = Ptr.GetSharedReferenceCount();
```

---

## TSharedRef

null にならない shared_ptr。関数の引数・戻り値で「null でないことを型で保証」したい場合:

```cpp
TSharedRef<FMyData> Ref = MakeShared<FMyData>();

// TSharedPtr との相互変換
TSharedPtr<FMyData> Ptr = Ref;           // TSharedRef → TSharedPtr（常に成功）
TSharedRef<FMyData> Ref2 = Ptr.ToSharedRef();  // TSharedPtr → TSharedRef（null なら assert）

// TSharedRef は null チェック不要
Ref->DoSomething();  // IsValid() 不要
```

---

## TWeakPtr

所有権を持たない弱参照。循環参照の回避・キャッシュ・ライフタイム追跡に使う:

```cpp
TSharedPtr<FMyData> Owner = MakeShared<FMyData>();

// 弱参照を取得
TWeakPtr<FMyData> Weak = Owner;

// 使う前に Lock して有効性確認
if (TSharedPtr<FMyData> Locked = Weak.Pin())
{
    // この Locked が生きている間、Owner は delete されない
    Locked->DoSomething();
}

// 有効性確認（Lock なし）
bool bValid = Weak.IsValid();

// 解放後は IsValid が false になる
Owner.Reset();
bool bNowInvalid = Weak.IsValid();  // false
```

---

## TUniquePtr

単独所有。コピー不可・ムーブ専用。`std::unique_ptr<T>` 相当:

```cpp
#include "Templates/UniquePtr.h"

TUniquePtr<FMyData> Ptr = MakeUnique<FMyData>(42);

// アクセス
Ptr->DoSomething();

// 所有権移転（ムーブ）
TUniquePtr<FMyData> Moved = MoveTemp(Ptr);  // Ptr は null になる

// 解放
Ptr.Reset();

// 生ポインタ取得（所有権は渡さない）
FMyData* Raw = Ptr.Get();

// 所有権を放棄して生ポインタを返す（delete の責任は呼び出し元に）
FMyData* Released = Ptr.Release();
```

---

## スレッド安全モード（ESPMode）

デフォルトは `ESPMode::ThreadSafe`（アトミックな参照カウント）。シングルスレッド確実なら `ESPMode::NotThreadSafe` でオーバーヘッド削減:

```cpp
// スレッドセーフ（デフォルト）
TSharedPtr<FMyData, ESPMode::ThreadSafe> SafePtr = MakeShared<FMyData>();

// 非スレッドセーフ（GameThread 専用など）
TSharedPtr<FMyData, ESPMode::NotThreadSafe> FastPtr = MakeShared<FMyData, ESPMode::NotThreadSafe>();
```

---

## UObject との使い分け

```
非 UObject クラス → TSharedPtr / TUniquePtr
UObject 派生クラス → UPROPERTY + 生ポインタ（GC が管理）
                     または TWeakObjectPtr（弱参照）
```

```cpp
// UObject 向けの弱参照
TWeakObjectPtr<AMyActor> WeakActor = SomeActor;

if (WeakActor.IsValid())
{
    WeakActor->DoSomething();
}
```

`TSharedPtr<UObject>` は **使ってはいけない**。GC と参照カウントの二重管理でリークやクラッシュの原因になる。

---

## TOptional — null を持てる値型

ポインタを使わず「値があるかもしれない」を表現:

```cpp
#include "Misc/Optional.h"

TOptional<int32> MaybeValue;

// 値をセット
MaybeValue = 42;

// チェックと取得
if (MaybeValue.IsSet())
{
    int32 Val = MaybeValue.GetValue();
    // または
    int32 Val2 = *MaybeValue;
}

// デフォルト値付き取得
int32 WithDefault = MaybeValue.Get(0);

// リセット
MaybeValue.Reset();

// 関数の戻り値として
TOptional<float> FindMaxValue(const TArray<float>& Arr)
{
    if (Arr.IsEmpty()) return TOptional<float>();  // 無効
    return *Algo::MaxElement(Arr);                 // 有効
}
```

---

## 選択ガイド

```
オブジェクトの所有が 1 箇所
  └─ TUniquePtr<T>

複数箇所で共有所有
  └─ TSharedPtr<T> / TSharedRef<T>

所有はせず「まだ生きているか」を確認したい
  ├─ 非 UObject → TWeakPtr<T>
  └─ UObject   → TWeakObjectPtr<T>

値が「あるかもしれない」を型で表現
  └─ TOptional<T>

UObject 派生
  └─ UPROPERTY + 生ポインタ（GC に任せる）
```

---

## コード実行フロー

### エントリポイント（MakeShared 〜 参照カウント 〜 Pin 〜 解放）

```
(MakeShared - 一括確保)
TSharedPtr<FMyData> P = MakeShared<FMyData>(Args...)              [SharedPointer.h]
  └─ FSharedReferencer<Mode>::MakeSharedRef
       └─ TIntrusiveReferenceController<T, Mode>* Ref
            ├─ ::operator new(sizeof(TIntrusiveReferenceController))  ← 1回のアロケで
            ├─ new (RefCtrl) TIntrusiveReferenceController(Args...)   ← Ctrl + Object 同居
            │    ├─ SharedRefCount = 1
            │    ├─ WeakRefCount   = 1                                ← 弱参照分も最初から
            │    └─ T(Args...)                                        ← in-place 構築
            └─ return TSharedRef<T>(Ref)

(MakeShareable - 旧式 2 回アロック)
TSharedPtr<FMyData> P = MakeShareable(new FMyData())              [SharedPointer.h]
  └─ FSharedReferencer<Mode>::FSharedReferencer(NewPtr)
       └─ new TReferenceControllerWithDeleter(NewPtr)             ← Object とは別ヒープ
            ← 2 回アロケのため Cache miss しやすい

(コピー - 参照カウント増)
TSharedPtr<T> Q = P                                                [SharedPointer.h]
  └─ FSharedReferencer::FSharedReferencer(Other)
       └─ ReferenceController->AddSharedReference()
            └─ if (ThreadSafe) FPlatformAtomics::InterlockedIncrement(&SharedRefCount)
            └─ else            ++SharedRefCount                      ← NotThreadSafe は単純 inc

(解放 - 参照カウント減)
P.Reset() または スコープ終了                                       [SharedPointer.h]
  └─ ~FSharedReferencer()
       └─ ReferenceController->ReleaseSharedReference()
            ├─ if (--SharedRefCount == 0):
            │    ├─ DestroyObject() → T::~T()                       ← オブジェクト破棄
            │    └─ ReleaseWeakReference()                          ← 弱参照分も減
            │         └─ if (--WeakRefCount == 0):
            │              └─ delete this (Controller 解放)         ← 全てが消えた

(TWeakPtr::Pin - 安全な強参照取得)
TSharedPtr<T> Locked = Weak.Pin()                                 [SharedPointer.h]
  └─ FWeakReferencer::Pin()
       └─ ReferenceController->ConditionallyAddSharedReference()
            ├─ Loop CAS:
            │    ├─ Old = SharedRefCount
            │    ├─ if (Old == 0) return false                       ← 既に破棄済み
            │    └─ CompareAndSwap(SharedRefCount, Old, Old + 1)    ← 競合なら再試行
            └─ return TSharedPtr (有効) または empty

(TUniquePtr - 軽量単独所有)
TUniquePtr<T> U = MakeUnique<T>(Args...)                          [UniquePtr.h]
  └─ TUniquePtr(new T(Args...))                                    ← 単純 new、参照カウントなし
~TUniquePtr()
  └─ delete Ptr                                                    ← 単純 delete

(MoveTemp - 所有権移転)
TUniquePtr<T> Moved = MoveTemp(U)
  └─ Moved.Ptr = U.Ptr; U.Ptr = nullptr                            ← ポインタすり替えのみ
```

### フロー詳細

1. **MakeShared の最適化** — 参照カウントブロックとオブジェクトを単一アロケーションで隣接配置（`TIntrusiveReferenceController<T>`）。`MakeShareable(new T)` は 2 回アロケートしてしまうので推奨されない。
2. **WeakRefCount の初期値** — `MakeShared` 直後の `WeakRefCount` は 1（強参照ブロック自身が弱参照を 1 つ持つ扱い）。これにより `SharedRefCount==0` でもコントローラ自体は `WeakRefCount==0` まで残り、`TWeakPtr::Pin` の競合を安全にハンドルできる。
3. **ThreadSafe vs NotThreadSafe** — `ESPMode::ThreadSafe`（既定）は `FPlatformAtomics::InterlockedIncrement` を使う。`NotThreadSafe` は単純な `++` で速いが GameThread 限定。Slate ウィジェット系は `ThreadSafe`、エディタ拡張の一部は `NotThreadSafe` を選ぶ。
4. **Pin の CAS ループ** — `TWeakPtr::Pin` は `ConditionallyAddSharedReference` で CAS ループ。「`SharedRefCount > 0` ならインクリメント」を原子的に行うことで、別スレッドが同時に最後の `Reset` をしても安全。
5. **TUniquePtr が軽い理由** — 参照カウントを持たず、ポインタ 1 本だけ。デストラクタは単純 `delete`。`MoveTemp` でポインタすり替えのみ。`std::unique_ptr` と同じ。
6. **UObject に使ってはいけない** — `TSharedPtr<UObject派生>` は GC（[[../../UObject/Details/b_garbage_collection]]）とは独立して参照カウントを管理してしまい、二重解放やリークの原因になる。UObject は `UPROPERTY` 生ポインタか `TWeakObjectPtr` を使う。
7. **TSharedRef の null 不可** — コンパイル時に `nullptr` を弾くため、関数引数・戻り値で「絶対に有効」を表現できる。`TSharedPtr → TSharedRef` 変換は実行時 assert で null チェック。
8. **TArray との組合せ** — `TArray<TSharedPtr<T>>` は要素ごとに参照カウントブロックがあり、リアロケーション時にコピー（参照カウント増減）が走る。`Reserve` で再確保を抑えるとよい（[[a_array_map_set]]）。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `MakeShared<T>` | `Templates/SharedPointer.h` | 参照ブロック + オブジェクト一括確保 |
| `TIntrusiveReferenceController` | `Templates/SharedPointerInternals.h` | カウンタ + オブジェクト同居型 |
| `TReferenceControllerWithDeleter` | `Templates/SharedPointerInternals.h` | 旧 MakeShareable 用（別アロック） |
| `FSharedReferencer::AddSharedReference` / `ReleaseSharedReference` | `Templates/SharedPointer.h` | 強参照カウント増減 |
| `FWeakReferencer::Pin` / `ConditionallyAddSharedReference` | `Templates/SharedPointer.h` | 弱参照→強参照の安全変換（CAS） |
| `MakeUnique<T>` / `TUniquePtr` | `Templates/UniquePtr.h` | 単独所有スマートポインタ |
| `MoveTemp` | `Templates/UnrealTemplate.h` | 所有権ムーブ |

---

## 関連ドキュメント

- [[a_array_map_set]] — TArray<TSharedPtr<T>> など
- [[../../UObject/Details/b_garbage_collection]] — GC と UObject ポインタ管理
- [[Reference/ref_containers_api]] — TSharedPtr / TWeakPtr / TUniquePtr API 詳細
