# リファレンス：RenderGraphAllocator.h / RenderGraphResourcePool.cpp

- グループ: b - Resources
- 上位: [[b_rdg_resources]]
- 関連: [[ref_rdg_builder]] | [[ref_rdg_resources]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphAllocator.h`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphAllocator.cpp`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphResourcePool.cpp`

## 概要

RDG グラフのライフタイムに紐付いたメモリアロケータ `FRDGAllocator` と、  
フレームをまたぐバッファプール `FRDGBufferPool` を定義する。

---

## FRDGAllocator

グラフ全体で使用するメモリを管理するスタックアロケータ。  
`FMemStackBase` ベース（通常ビルド）または `FMemory::Malloc`（ASan ビルド）で実装される。

```cpp
class FRDGAllocator
{
public:
    // スレッドローカルなグローバルインスタンスを取得
    static FRDGAllocator& GetTLS();

    // 生メモリを確保（デストラクタ追跡なし）
    void* Alloc(uint64 SizeInBytes, uint32 AlignInBytes);

    // POD 型を未初期化で確保
    template <typename PODType>
    PODType* AllocUninitialized(uint64 Count = 1);

    // オブジェクトを確保・構築し、デストラクタを追跡
    template <typename T, typename... TArgs>
    T* Alloc(TArgs&&... Args);

    // C++ オブジェクトをデストラクタ追跡なしで確保（危険 — 慎重に使うこと）
    template <typename T, typename... TArgs>
    T* AllocNoDestruct(TArgs&&... Args);

    // 現在の確保バイト数
    int32 GetByteCount() const;

    // 全確保済みメモリを解放
    void ReleaseAll();
};
```

### ネスト型

```cpp
// デストラクタ追跡用基底クラス
class FRDGAllocator::FObject
{
public:
    virtual ~FObject() = default;
};

// 型ごとのデストラクタラッパー
template <typename T>
class FRDGAllocator::TObject final : FObject
{
    T Alloc;
};
```

### コンパイル設定

```cpp
// ASan ビルドでは MemStack の代わりに Malloc を使用
#define RDG_USE_MALLOC USING_ADDRESS_SANITISER
// デバッグビルドではアクセス競合を検出
#define RDG_ALLOCATOR_DEBUG USING_ADDRESS_SANITISER || UE_BUILD_DEBUG
```

---

## FRDGAllocatorScope

TLS アロケータを一時的に差し替えるスコープクラス。  
ネストした RDG グラフや並列セットアップタスクで使用される。

```cpp
class FRDGAllocatorScope
{
public:
    FRDGAllocatorScope(FRDGAllocator& Allocator);  // TLS を差し替え
    ~FRDGAllocatorScope();                          // TLS を元に戻す
};
```

---

## コンテナアロケータ

```cpp
// RDG アロケータを使うコンテナアロケータ（TArray 等と組み合わせる）
template<uint32 Alignment = DEFAULT_ALIGNMENT>
class TRDGArrayAllocator { ... };

// よく使うエイリアス
using FRDGArrayAllocator    = TRDGArrayAllocator<>;
using FRDGBitArrayAllocator = TInlineAllocator<4, FRDGArrayAllocator>;
using FRDGSetAllocator      = TSetAllocator<FRDGSparseArrayAllocator, ...>;
```

---

## FRDGBufferPool（GRenderGraphResourcePool）

フレームをまたいで再利用されるバッファプール。  
`FRDGBuilder` 内で自動的に使用され、`QueueBufferExtraction` 後に `FRDGPooledBuffer` として外部から参照できる。

### バッファアライメント

```cpp
enum class ERDGPooledBufferAlignment
{
    None,        // アライメントなし（要求サイズのまま確保）
    PowerOfTwo,  // 2 の冪乗に切り上げ
    Page,        // 64 KB ページ境界に切り上げ
};
```

### グローバルインスタンス

```cpp
// グローバルなバッファプール（RenderGraphResourcePool.cpp で定義）
extern RENDERCORE_API FRenderGraphResourcePool GRenderGraphResourcePool;

// デバッグコマンド: バッファプールのメモリ使用量をダンプ
// コンソール: r.DumpBufferPoolMemory
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.DumpBufferPoolMemory` | — | バッファプールのメモリ使用量をコンソールに出力するコマンド |

---

## フレンドマクロ

```cpp
// FRDGAllocator が private コンストラクタを呼べるよう許可するマクロ
#define RDG_FRIEND_ALLOCATOR_FRIEND(Type) \
    friend class FRDGAllocator::TObject<Type>
```
