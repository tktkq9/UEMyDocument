# PCG 標準ノード全一覧（主要カテゴリ）

- 上位: [[PCG/01_overview]]
- ソース: `Engine/Plugins/PCG/Source/PCG/Public/Elements/`

---

## Sampler ノード

| ノード（Settings クラス） | ファイル | 説明 |
|--------------------------|---------|------|
| `UPCGSurfaceSamplerSettings` | `PCGSurfaceSampler.h` | サーフェス上にポイントを配置 |
| `UPCGSplineSamplerSettings` | `PCGSplineSampler.h` | スプライン上・内部にポイント配置 |
| `UPCGVolumeSamplerSettings` | `PCGVolumeSampler.h` | ボリューム内をランダムサンプリング |
| `UPCGTextureSamplerSettings` | `PCGTextureSampler.h` | テクスチャをポイント属性として取得 |

---

## Filter ノード

| ノード（Settings クラス） | ファイル | 説明 |
|--------------------------|---------|------|
| `UPCGAttributeFilterSettings` | `PCGAttributeFilter.h` | 属性値の条件でフィルタリング |
| `UPCGSelectPointsSettings` | `PCGSelectPoints.h` | サブセット選択 |
| `UPCGBooleanSelectSettings` | `ControlFlow/PCGBooleanSelect.h` | 条件分岐でデータ切り替え |
| `UPCGDensityFilterSettings` | — | 密度値でフィルタリング |

---

## Transform ノード

| ノード（Settings クラス） | ファイル | 説明 |
|--------------------------|---------|------|
| `UPCGBoundsModifierSettings` | `PCGBoundsModifier.h` | バウンドの変更（Set/Scale/Intersect）|
| `UPCGApplyScaleToBoundsSettings` | `PCGApplyScaleToBounds.h` | スケールをバウンドに適用 |
| `UPCGCopyPointsSettings` | `PCGCopyPoints.h` | ポイントのコピー・変換 |
| `UPCGCollapsePointsSettings` | `PCGCollapsePoints.h` | ポイントの結合 |
| `UPCGCombinePointsSettings` | `PCGCombinePoints.h` | 複数ポイントセットの統合 |

---

## Spawner ノード

| ノード（Settings クラス） | ファイル | 説明 |
|--------------------------|---------|------|
| `UPCGStaticMeshSpawnerSettings` | `PCGStaticMeshSpawner.h` | StaticMesh を ISM でスポーン（主力） |
| `UPCGSkinnedMeshSpawnerSettings` | `PCGSkinnedMeshSpawner.h` | SkinnedMesh をスポーン |
| `UPCGCreateTargetActorSettings` | `PCGCreateTargetActor.h` | アクタを動的生成 |

---

## Data 取得ノード

| ノード | 説明 |
|-------|------|
| Get Actor Data | アクタのコンポーネントからデータ取得 |
| Get Landscape Data | ランドスケープの高さ・法線・レイヤー |
| Get Spline Data | スプラインコンポーネントのデータ |
| Get Volume Data | ボックスボリュームのデータ |
| Get Actor Properties | アクタのプロパティを属性として取得 |

---

## Metadata / Attribute ノード

| ノード（Settings クラス） | ファイル | 説明 |
|--------------------------|---------|------|
| `UPCGAttributeTransferSettings` | `PCGAttributeTransferElement.h` | 最近傍ポイントへの属性転送 |
| `UPCGAttributeRemap` | `PCGAttributeRemap.h` | 属性値の再マッピング |
| `UPCGAttributeCastSettings` | `PCGAttributeCast.h` | 属性の型変換 |
| `UPCGAttributeNoiseSettings` | `PCGAttributeNoise.h` | 属性値にノイズ付加 |
| `UPCGAttributeReduceSettings` | `PCGAttributeReduceElement.h` | 属性の集約（Sum/Min/Max/Average） |
| `UPCGAttributeRemoveDuplicatesSettings` | `PCGAttributeRemoveDuplicates.h` | 重複属性値の除去 |

---

## ControlFlow ノード

| ノード（Settings クラス） | ファイル | 説明 |
|--------------------------|---------|------|
| `UPCGBranchSettings` | `ControlFlow/PCGBranch.h` | Bool 値による分岐 |
| `UPCGSwitchSettings` | `ControlFlow/PCGSwitch.h` | 複数パスの切り替え |
| `UPCGMultiSelectSettings` | `ControlFlow/PCGMultiSelect.h` | 複数の出力選択 |
| `UPCGWaitSettings` | `ControlFlow/PCGWait.h` | 並列タスクの同期待ち |
| `UPCGQualityBranchSettings` | `ControlFlow/PCGQualityBranch.h` | スケーラビリティ設定による分岐 |

---

## Subgraph ノード

| ノード | 説明 |
|-------|------|
| Subgraph（UPCGSubgraphSettings） | 別の PCGGraph を呼び出す（関数呼び出し相当） |
| Dynamic Subgraph | ランタイムでグラフを切り替え |
| Loop | 指定回数繰り返し |

---

## Physics / World ノード

| ノード | 説明 |
|-------|------|
| Project Points on Terrain | ポイントを地形に投影 |
| Raycast | レイキャストで位置取得 |
| Get World Ray Hit | ワールドに対してレイを飛ばす |

---

## Debug ノード

| ノード | 説明 |
|-------|------|
| Debug Print（PCGDebugSettings） | ポイント情報をデバッグ表示 |
| Attribute Debug Output | 属性値を出力 |

---

## GPU 対応ノード

以下のノードは `DisplayExecuteOnGPUSetting() = true` で GPU 実行オプションを持つ：

- `UPCGStaticMeshSpawnerSettings` — メッシュスポーン
- `UPCGCopyPointsSettings` — ポイントコピー
- `UPCGSurfaceSamplerSettings` — サーフェスサンプリング

GPU 実行には Compute Shader が使用され、大量ポイントの処理高速化が期待できる。

---

## UPCGMeshSelectorBase 派生クラス

`UPCGStaticMeshSpawnerSettings` の `MeshSelectorParameters` で設定するメッシュ選択方式。

| クラス | 選択方式 |
|-------|---------|
| `UPCGMeshSelectorWeighted` | ウェイト付きランダム選択 |
| `UPCGMeshSelectorWeightedByCategory` | カテゴリ属性で分類してランダム |
| `UPCGMeshSelectorByAttribute` | 属性値で直接メッシュを指定 |

---

## 参考：ノード一覧の確認方法

エディタの PCG グラフ内で右クリック → **ノードリスト** から全標準ノードを確認できる。また `EPCGSettingsType` でカテゴリ分類されているため、C++ で `GetType()` をチェックすることで種別を判別できる。
