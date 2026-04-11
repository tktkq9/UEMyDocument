# リファレンス：RenderGraphAllocator.h / RenderGraphAllocator.cpp / RenderGraphResourcePool.cpp

- グループ: b - Resources
- 上位: [[b_rdg_resources]]
- 関連: [[ref_rdg_builder]] | [[ref_rdg_resources]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphAllocator.h`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphAllocator.cpp`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphResourcePool.cpp`

## 概要

RDG グラフのライフタイムに紐付くメモリアロケータ `FRDGAllocator` と、  
フレームをまたいでバッファを再利用するプール `FRDGBufferPool` を定義する。

---

## FRDGAllocator

グラフ内の CPU メモリを管理するスタックアロケータ。  
グラフの `Execute()` 完了後に全メモリが一括解放される。

```
通常ビルド:   FMemStackBase（高速なスタック割り当て）
ASan ビルド:  FMemory::Malloc（メモリサニタイザと互換）
```

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `MemStack` | `FMemStackBase` | 通常ビルドのスタックメモリ（`RDG_USE_MALLOC` が 0 の場合） |
| `Mallocs` | `TArray<void*>` | ASan ビルドで確保したポインタ一覧 |
| `NumMallocBytes` | `uint64` | ASan ビルドの総確保バイト数 |
| `Objects` | `TArray<FObject*>` | デストラクタ追跡対象のオブジェクト一覧 |

### 主要関数

```cpp
// ─── 生メモリ確保（デストラクタ追跡なし）───
void* Alloc(uint64 SizeInBytes, uint32 AlignInBytes);

// ─── POD 型の未初期化確保（デストラクタ追跡なし）───
template <typename PODType>
PODType* AllocUninitialized(uint64 Count = 1);

// ─── C++ オブジェクトの確保・構築（デストラクタを FObject 経由で追跡）───
template <typename T, typename... TArgs>
T* Alloc(TArgs&&... Args);

// ─── デストラクタ追跡なしの C++ オブジェクト確保（危険）───
template <typename T, typename... TArgs>
T* AllocNoDestruct(TArgs&&... Args);

// ─── 統計 ───
int32 GetByteCount() const;

// ─── 全解放 ───
void ReleaseAll();

// ─── TLS インスタンス取得 ───
static FRDGAllocator& GetTLS();
```

### 内部処理フロー（ReleaseAll）

```
ReleaseAll()
  ├─ Objects 配列を逆順に Delete（デストラクタを呼ぶ）
  ├─ Objects.Empty()
  └─ [ASan] Mallocs を全 FMemory::Free → NumMallocBytes = 0
     [通常] MemStack.Flush()
```

### 使用箇所
- [[ref_rdg_builder]] の `FRDGBuilder` — グラフ全体のメモリを管理
- `FRDGBuilder::Alloc()` / `AllocObject()` / `AllocParameters()` の実体

---

## FRDGAllocatorScope

TLS アロケータを一時的に差し替えるスコープクラス。

```cpp
class FRDGAllocatorScope
{
public:
    FRDGAllocatorScope(FRDGAllocator& Allocator);  // TLS を InAllocator に差し替え
    ~FRDGAllocatorScope();                          // TLS を元のアロケータに戻す

private:
    void* AllocatorToRestore;  // 差し替え前の TLS ポインタ
};
```

### 使用箇所
- RDG ビルダー内部で並列セットアップタスクを起動するとき

---

## コンテナアロケータ

```cpp
// RDG アロケータを使う TArray 用アロケータ
template<uint32 Alignment = DEFAULT_ALIGNMENT>
class TRDGArrayAllocator { ... };

// よく使うエイリアス
using FRDGArrayAllocator    = TRDGArrayAllocator<>;
using FRDGBitArrayAllocator = TInlineAllocator<4, FRDGArrayAllocator>;
using FRDGSparseArrayAllocator = TSparseArrayAllocator<FRDGArrayAllocator, FRDGBitArrayAllocator>;
using FRDGSetAllocator = TSetAllocator<FRDGSparseArrayAllocator, TInlineAllocator<1, FRDGBitArrayAllocator>>;
```

### ResizeAllocation 内部

```cpp
// TRDGArrayAllocator は ResizeAllocation で FRDGAllocator::GetTLS() からメモリを取得する
void ResizeAllocation(SizeType CurrentNum, SizeType NewMax, SIZE_T NumBytesPerElement)
{
    const int32 AllocSize = NewMax * NumBytesPerElement;
    const int32 AllocAlignment = FMath::Max(Alignment, (uint32)alignof(ElementType));
    Data = (ElementType*)FRDGAllocator::GetTLS().Alloc(AllocSize, AllocAlignment);
    if (OldData && CurrentNum)
        FMemory::Memcpy(Data, OldData, NumCopiedElements * NumBytesPerElement);
}
```

---

## FRDGPooledBuffer

フレームをまたいで再利用されるバッファ。  
`FRDGBufferPool::ScheduleAllocation()` でプールから取得し、  
`QueueBufferExtraction()` で外部コードに公開される。

```cpp
class FRDGPooledBuffer : public FRefCountBase
{
public:
    FRHIBuffer*            GetRHI()           const;
    FRHIShaderResourceView* GetSRV()          const;   // 全体 SRV
    uint64                 GetAlignedSize()   const;
    uint32                 NumAllocatedElements;        // 実際に確保した要素数
    const char*            Name;                        // デバッグ名

    // View キャッシュ（SRV/UAV を再生成せず再利用）
    FRHIBufferViewCache ViewCache;
};
```

### バッファプール（FRDGBufferPool / GRenderGraphResourcePool）

```
FRDGBufferPool::ScheduleAllocation()
  ├─ 既存プールを desc + hash でスキャン（TryFindPooledBuffer）
  │   └─ 参照カウントが 1（プールのみ保持）で desc が一致 → 再利用
  └─ 見つからない場合 → CreateBuffer() で新規 RHI バッファを確保
```

### バッファアライメント

```cpp
enum class ERDGPooledBufferAlignment
{
    None,        // 要求サイズのまま確保
    PowerOfTwo,  // 2 の冪乗に切り上げ
    Page,        // 64 KB ページ境界に切り上げ（PowerOfTwo + Page 丸め）
};
```

### デバッグコマンド

| コマンド | 説明 |
|---------|------|
| `r.DumpBufferPoolMemory` | プール内バッファの一覧（サイズ・名前・未使用フレーム数）をコンソールに出力 |

### 使用箇所
- [[ref_rdg_builder]] `FRDGBuilder::QueueBufferExtraction()` の戻り値
- `FRDGBuilder::GetPooledBuffer()` で外部から RHI バッファを取得

---

## コンパイル設定マクロ

```cpp
// ASan 有効時: malloc/free ベースに切り替え
#define RDG_USE_MALLOC USING_ADDRESS_SANITISER

// ASan または DEBUG ビルド: アクセス競合の検出を有効化
#define RDG_ALLOCATOR_DEBUG USING_ADDRESS_SANITISER || UE_BUILD_DEBUG
```

---

## フレンドマクロ

```cpp
// FRDGAllocator の TObject<T> を private コンストラクタから生成可能にする
#define RDG_FRIEND_ALLOCATOR_FRIEND(Type) \
    friend class FRDGAllocator::TObject<Type>
```

### 使用箇所
- `FRDGTexture`, `FRDGBuffer` 等の private 宣言内
