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

## 関連ドキュメント

- [[a_array_map_set]] — TArray<TSharedPtr<T>> など
- [[../../UObject/Details/b_garbage_collection]] — GC と UObject ポインタ管理
- [[Reference/ref_containers_api]] — TSharedPtr / TWeakPtr / TUniquePtr API 詳細
