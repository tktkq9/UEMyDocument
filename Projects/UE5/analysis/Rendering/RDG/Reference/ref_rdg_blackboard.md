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

```cpp
class FRDGBlackboard
{
public:
    // 新規作成（すでに存在する場合はアサート）
    template <typename StructType, typename... ArgsType>
    StructType& Create(ArgsType&&... Args);

    // Mutable 取得（存在しない場合は nullptr）
    template <typename StructType>
    StructType* GetMutable() const;

    // Const 取得（存在しない場合は nullptr）
    template <typename StructType>
    const StructType* Get() const;

    // Mutable 取得（存在しない場合はアサート）
    template <typename StructType>
    StructType& GetMutableChecked() const;

    // Const 取得（存在しない場合はアサート）
    template <typename StructType>
    const StructType& GetChecked() const;

    // 存在しなければ作成して返す
    template <typename StructType, typename... ArgsType>
    StructType& GetOrCreate(ArgsType&&... Args);

private:
    FRDGAllocator& Allocator;                     // グラフアロケータへの参照
    TArray<FStruct*, FRDGArrayAllocator> Blackboard;  // 型インデックス → インスタンスの配列
    friend class FRDGBuilder;
};
```

---

## RDG_REGISTER_BLACKBOARD_STRUCT マクロ

Blackboard で使用する構造体を登録するためのマクロ。  
型と一意のインデックスを関連付ける。

```cpp
// ヘッダまたは .cpp で 1 回だけ宣言する
RDG_REGISTER_BLACKBOARD_STRUCT(FMyStruct)
```

マクロが展開すると `FRDGBlackboard::GetTypeName<FMyStruct>()` の特殊化が生成される。  
登録なしに `Create<FMyStruct>()` を呼ぶとコンパイルエラーになる。

---

## 使用パターン

### 書き込み（パス A）

```cpp
// 構造体を Blackboard に登録
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
const auto* Shared = GraphBuilder.Blackboard.Get<FSceneSharedTextures>();
if (Shared) { ... }
```

---

## 内部実装メモ

- 各型に静的な一意インデックスが `AllocateIndex()` で割り振られる
- 配列の添字は型インデックス（= 初回アクセス時に採番）
- インデックスはプロセス起動中は不変（型が追加されると配列が拡張される）
