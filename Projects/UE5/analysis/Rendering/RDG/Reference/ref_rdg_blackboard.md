# リファレンス：RenderGraphBlackboard.h / RenderGraphBlackboard.cpp

- グループ: d - Parameters
- 上位: [[d_rdg_parameters]]
- 関連: [[ref_rdg_builder]] | [[ref_rdg_parameter]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphBlackboard.h`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphBlackboard.cpp`

## 概要

同一フレーム内の異なるパス間でデータを型安全に受け渡すための辞書クラス。  
`FRDGBuilder::Blackboard` として公開される。`Execute()` 後に自動破棄される。

**適切な用途:** パイプライン全体にまたがる共有テクスチャ参照（SceneColor 等）。  
**不適切な用途:** 局所的なデータの受け渡し（関数引数を使うべき）。

---

## FRDGBlackboard

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Allocator` | `FRDGAllocator&` | グラフアロケータへの参照。登録インスタンスはこのアロケータから確保される |
| `Blackboard` | `TArray<FStruct*, FRDGArrayAllocator>` | 型インデックス → 仮想基底インスタンスの配列（null = 未登録） |

### 内部型

```cpp
// 仮想基底クラス（デストラクタ追跡用）
struct FStruct { virtual ~FStruct() = default; };

// 型付き具体クラス
template <typename StructType>
struct TStruct final : public FStruct
{
    StructType Struct;  // ユーザー定義の構造体インスタンス
};
```

### 公開メソッド

```cpp
// 新規作成（すでに存在する場合は checkf でアサート）
template <typename StructType, typename... ArgsType>
StructType& Create(ArgsType&&... Args);

// Mutable 取得（存在しない場合は nullptr）
template <typename StructType>
StructType* GetMutable() const;

// Const 取得（存在しない場合は nullptr）
template <typename StructType>
const StructType* Get() const;

// Mutable 取得（存在しない場合は checkf でアサート）
template <typename StructType>
StructType& GetMutableChecked() const;

// Const 取得（存在しない場合は checkf でアサート）
template <typename StructType>
const StructType& GetChecked() const;

// 存在しなければ作成して返す
template <typename StructType, typename... ArgsType>
StructType& GetOrCreate(ArgsType&&... Args);
```

### 内部処理フロー（Create）

```
Create<StructType>(Args...)
  ├─ GetStructIndex<StructType>()
  │   └─ 初回呼び出し時: AllocateIndex() でグローバル一意インデックスを割り当て
  │       └─ 以降は static uint32 Index にキャッシュ（= 再割り当てなし）
  ├─ Blackboard 配列が足りない場合: SetNumZeroed で拡張
  ├─ Blackboard[StructIndex] が非 null なら checkf アサート
  └─ Allocator.Alloc<TStruct<StructType>>(Args...) で確保・構築
     └─ Blackboard[StructIndex] に登録、struct の参照を返す
```

### 使用箇所
- [[ref_rdg_builder]] `FRDGBuilder::Blackboard` — パブリックメンバとして公開
- `DeferredShadingRenderer.cpp` — `FSceneTexturesConfig` / `FSceneTextures` の共有

---

## RDG_REGISTER_BLACKBOARD_STRUCT マクロ

Blackboard で使用する構造体を登録するためのマクロ。  
型と一意のインデックスを関連付ける。

```cpp
// マクロの展開内容
#define RDG_REGISTER_BLACKBOARD_STRUCT(StructType)          \
    template <>                                              \
    inline FString FRDGBlackboard::GetTypeName<StructType>()\
    {                                                        \
        return GetTypeName(TEXT(#StructType), TEXT(__FILE__), __LINE__); \
    }
```

登録なしに `Create<MyStruct>()` を呼ぶと、`static_assert(sizeof(StructType) == 0)` が発動してコンパイルエラーになる。

---

## 使用パターン

### 書き込み（パス A）

```cpp
// ヘッダで構造体を定義・登録
struct FSceneSharedTextures
{
    FRDGTextureRef SceneColor = nullptr;
    FRDGTextureRef SceneDepth = nullptr;
};
RDG_REGISTER_BLACKBOARD_STRUCT(FSceneSharedTextures)

// パス A でデータを作成・格納
auto& Shared = GraphBuilder.Blackboard.Create<FSceneSharedTextures>();
Shared.SceneColor = SceneColorTexture;
Shared.SceneDepth = SceneDepthTexture;
```

### 読み込み（パス B）

```cpp
// パス B でデータを取得（存在しない場合はアサート）
const auto& Shared = GraphBuilder.Blackboard.GetChecked<FSceneSharedTextures>();
FRDGTextureRef Color = Shared.SceneColor;

// 存在しない可能性がある場合
const auto* SharedPtr = GraphBuilder.Blackboard.Get<FSceneSharedTextures>();
if (SharedPtr) { ... }
```

---

> [!note]- 型インデックスの割り当て方式（静的 once-initialization）
> `GetStructIndex<T>()` は `static uint32 Index = UINT_MAX` をスレッドセーフに初期化する。  
> `AllocateIndex()` はグローバルな原子カウンタをインクリメントして一意番号を返す。  
> インデックスはプロセス起動後に単調増加し、型の登録順に依存するが不変。  
> `Blackboard` 配列は必要に応じて `SetNumZeroed` で拡張されるため、初期サイズへの事前決定は不要。

> [!note]- GetOrCreate の使いどころ
> `GetOrCreate<T>()` はパスが複数の呼び出しパスから到達できる場合に安全に使える。  
> 最初の呼び出し時のみ `Create` を実行し、以降は `GetMutable` でキャッシュされた参照を返す。  
> ただし `Create` と `GetOrCreate` を混在させると 2 回目の `Create` でアサートするため、  
> どちらを使うかのルールをシステム内で統一することが重要。
