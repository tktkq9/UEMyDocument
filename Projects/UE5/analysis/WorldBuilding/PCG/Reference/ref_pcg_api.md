# UPCGComponent / UPCGGraph API リファレンス

- 上位: [[PCG/01_overview]]
- ソース: `Engine/Plugins/PCG/Source/PCG/Public/PCGComponent.h`
          `Engine/Plugins/PCG/Source/PCG/Public/PCGGraph.h`
          `Engine/Plugins/PCG/Source/PCG/Public/PCGSubsystem.h`

---

## UPCGComponent

アクタに付与して PCG グラフを実行するコンポーネント。`UActorComponent` 派生。

```cpp
UCLASS(BlueprintType, ClassGroup = (Procedural))
class UPCGComponent : public UActorComponent
```

### 生成トリガーの種類

```cpp
UENUM(Blueprintable)
enum class EPCGComponentGenerationTrigger : uint8
{
    GenerateOnLoad,    // コンポーネントロード時に自動生成
    GenerateOnDemand,  // 明示的な呼び出し時のみ生成
    GenerateAtRuntime, // ランタイム生成スケジューラから生成
};
```

### BP 公開メソッド

| メソッド | 説明 |
|---------|------|
| `Generate(bool bForce)` | ネットワーク複製あり生成 |
| `Cleanup(bool bRemoveComponents)` | ネットワーク複製ありクリーンアップ |
| `GenerateLocal(bool bForce)` | ローカルのみ生成（複製なし） |
| `CleanupLocal(bool bRemoveComponents)` | ローカルのみクリーンアップ |
| `SetGraph(UPCGGraphInterface*)` | グラフの変更（複製あり） |
| `NotifyPropertiesChangedFromBlueprint()` | プロパティ変更通知 → 再生成トリガー |
| `GetGeneratedGraphOutput() const` | 生成結果データの取得 |
| `ClearPCGLink(UClass*)` | 生成リソースを別アクタに移動 |
| `AddToManagedResources(UPCGManagedResource*)` | 管理リソース追加 |
| `AddComponentsToManagedResources(TArray<UActorComponent*>)` | コンポーネントを管理下に |
| `AddActorsToManagedResources(TArray<AActor*>)` | アクタを管理下に |

### 主要プロパティ

| プロパティ | 型 | 説明 |
|-----------|-----|------|
| `GraphInterface` | `UPCGGraphInterface*` | 実行するグラフ |
| `InputType` | `EPCGComponentInput` | 入力タイプ（Actor/Landscape/Other） |
| `GenerationTrigger` | `EPCGComponentGenerationTrigger` | 生成タイミング |
| `bActivated` | `bool` | コンポーネントが有効か |

### デリゲート（BP アサイン可）

```cpp
UPROPERTY(BlueprintAssignable)
FOnPCGGraphStartGeneratingExternal OnPCGGraphStartGenerating;

UPROPERTY(BlueprintAssignable)
FOnPCGGraphGeneratedExternal OnPCGGraphGenerated;

UPROPERTY(BlueprintAssignable)
FOnPCGGraphCancelledExternal OnPCGGraphCancelled;

UPROPERTY(BlueprintAssignable)
FOnPCGGraphCleanedExternal OnPCGGraphCleaned;
```

---

## UPCGGraph

```cpp
UCLASS(BlueprintType, ClassGroup = (Procedural))
class UPCGGraph : public UPCGGraphInterface
```

### BP 公開メソッド

| メソッド | 説明 |
|---------|------|
| `AddNodeOfType(Class, OutSettings)` | 指定クラスのノードを追加 |
| `AddNodeInstance(UPCGSettings*)` | 設定インスタンスでノード追加 |
| `AddNodeCopy(Settings, OutCopy)` | 設定をコピーしてノード追加 |
| `RemoveNode(UPCGNode*)` | ノード削除 |
| `RemoveNodes(TArray<UPCGNode*>)` | 複数ノード一括削除 |
| `AddEdge(From, FromPin, To, ToPin)` | エッジ追加 |
| `RemoveEdge(From, FromPin, To, ToPin)` | エッジ削除 |
| `GetInputNode() const` | 入力ノード取得 |
| `GetOutputNode() const` | 出力ノード取得 |
| `GetNodes() const` | 全ノード取得 |

### グラフパラメーター（C++ テンプレート版）

```cpp
// 取得
auto Result = GraphInterface->GetGraphParameter<float>(TEXT("Density"));
if (Result.HasValue()) { float D = Result.GetValue(); }

// 設定
GraphInterface->SetGraphParameter(TEXT("Density"), 0.5f);

// 配列パラメーターの更新
GraphInterface->UpdateArrayGraphParameter(TEXT("MeshList"),
    [](FPropertyBagArrayRef& Ref) -> bool {
        // 配列を操作...
        return true;
    });
```

---

## UPCGGraphInterface

```cpp
UCLASS(BlueprintType, Abstract, ClassGroup = (Procedural))
class UPCGGraphInterface : public UObject
```

| メソッド | 説明 |
|---------|------|
| `GetMutablePCGGraph()` | グラフ取得（変更可能） |
| `GetConstPCGGraph() const` | グラフ取得（読み取り専用） |
| `IsInstance() const` | グラフインスタンスか |

---

## UPCGSubsystem

PCG のグローバル管理サブシステム。`GetWorld()->GetSubsystem<UPCGSubsystem>()` で取得。

### 主要メソッド

```cpp
// PCGComponent の登録・更新
void RegisterOrUpdatePCGComponent(UPCGComponent* InComponent, bool bDoActorMapping = true);
void UnregisterPCGComponent(UPCGComponent* InComponent);

// グラフキャッシュの管理
void NotifyGraphChanged(UPCGGraph* InGraph);
void NotifyGraphInstanceChanged(UPCGGraphInstance* InGraphInstance);

// デバッグ
bool IsDebugging() const;
```

---

## UPCGNode — ノード操作

```cpp
UCLASS(MinimalAPI, ClassGroup = (Procedural))
class UPCGNode : public UObject
```

| メソッド | BP公開 | 説明 |
|---------|--------|------|
| `AddEdgeTo(FromPin, ToNode, ToPin)` | Yes | エッジ追加 |
| `RemoveEdgeTo(FromPin, ToNode, ToPin)` | Yes | エッジ削除 |
| `GetGraph() const` | Yes | 所属グラフ取得 |
| `GetNodeTitle(Type) const` | No | ノードタイトル取得 |

---

## 使用例（C++）

```cpp
// PCGComponent でグラフを実行
void AMyActor::GeneratePCG()
{
    UPCGComponent* PCGComp = FindComponentByClass<UPCGComponent>();
    if (!PCGComp) return;

    // グラフパラメーターを設定してから生成
    if (UPCGGraphInterface* Graph = PCGComp->GetGraph())
    {
        Graph->SetGraphParameter(TEXT("Density"), VegetationDensity);
    }

    // 非同期で生成（ネットワーク複製あり）
    PCGComp->Generate(/*bForce=*/true);
}

// 生成完了時のコールバック
void AMyActor::OnPCGGenerated(UPCGComponent* PCGComponent)
{
    // 生成されたデータにアクセス
    const FPCGDataCollection& Output = PCGComponent->GetGeneratedGraphOutput();
}
```
