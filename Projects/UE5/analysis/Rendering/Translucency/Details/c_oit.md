# c: Order-Independent Translucency（OIT）

- 対象ファイル: `OIT/OIT.h` / `OIT/OIT.cpp` / `OIT/OITParameters.h`
- 概要: [[20_translucency_overview]]

---

## 概要

通常の半透明描画は **Back-to-Front ソート**に依存するため、  
メッシュ内部のトライアングル順序が保証されない場合にアーティファクトが発生する。  
OIT（Order-Independent Translucency）はソート順に依存しない正確な合成を実現する。

---

## UE5 の OIT 実装方式

| 方式 | 説明 | CVar |
|-----|------|------|
| **SortedTriangles** | メッシュ単位のトライアングル GPU ソート | `r.OIT.SortedTriangles=1` |
| **SortedPixels (MLAB)** | ピクセル単位の Multi-Layer Alpha Blending | `r.OIT.SortedPixels=1` |

---

## SortedTriangles 方式

```
// トライアングルを GPU でソートして正しい描画順を確保

OIT::AddSortTrianglesPass(GraphBuilder, View, OITSceneData, BackToFront)
  │
  ├─ FOITSceneData::Allocate() で FSortedIndexBuffer を確保
  │   → FSortedIndexBuffer は UAV 対応インデックスバッファ
  │
  ├─ Sort CS でトライアングルをカメラ距離でソート
  │   Input:  SourceIndexBuffer（元のインデックス）
  │   Output: SortedIndexBuffer（ソート済みインデックス）
  │
  └─ 描画時は SortedIndexBuffer を使って RHIDrawIndexedPrimitive()

// 適用条件:
// OIT::IsCompatible(MeshBatch, FeatureLevel)
//   → LocalVertexFactory のみ対応
//   → メッシュに bSupportsOIT フラグが必要
```

---

## SortedPixels (MLAB) 方式

```
// MLAB: Multi-Layer Alpha Blending
// ピクセルごとに最大 N 個のサンプルを ROV (Rasterizer Ordered View) に記録し
// 最後に深度順にソートして合成する

[描画フェーズ]
OIT::CreateOITData(GraphBuilder, View, PassType)
  → FOITData を生成:
    SampleDataTexture:  Texture2DArray<uint>[W × H × MaxSamples]
                        → 各層に Color(R8G8B8A8 packed) + Depth(F16) を格納
    SampleCountTexture: Texture2D<uint>[W × H]
                        → ピクセルごとの使用サンプル数

[ピクセルシェーダー（半透明 BasePass）]
  FOITBasePassUniformParameters バインド:
    OutOITSampleCount: RasterizerOrderedTexture2D<uint>  // ROV で原子的書き込み
    OutOITSampleData:  RWTexture2DArray<uint>

  MLAB アルゴリズム（OITCombine.usf）:
    1. SampleCount = min(SampleCount + 1, MaxSamplePerPixel)
    2. 新サンプルを SampleData[Layer] に挿入
    3. Transmittance が TransmittanceThreshold(0.05) 以下なら早期終了
       → それ以降のサンプルは不要（不透明に近い）

[合成フェーズ]
OIT::AddOITComposePass(GraphBuilder, View, OITData, SceneColorTexture)
  → FOITPixelCombineCS ディスパッチ（8×8 スレッド）
  → 各ピクセルで:
      for layer in [0, SampleCount):
        Color = UnpackColor(SampleData[layer])
        Depth = UnpackDepth(SampleData[layer])
      → 深度昇順にソート
      → Back-to-Front で Alpha ブレンド
      → 最終色を OutColorTexture に書き込み
```

---

## OIT パス種別（EOITPassType）

```cpp
// OIT.h
enum EOITPassType
{
    OITPass_None = 0,
    OITPass_RegularTranslucency  = 1,  // TPT_StandardTranslucency
    OITPass_SeperateTranslucency = 2   // TPT_TranslucencyAfterDOF
};
// r.OIT.SortedPixels.PassType = 3 で両方有効（デフォルト）
```

---

## FOITSceneData と FSortedIndexBuffer

```cpp
// OIT.h
struct FOITSceneData
{
    TArray<FSortedTriangleData> Allocations;    // 確保済みトライアングルデータ
    TArray<FSortedIndexBuffer*> FreeBuffers;    // 再利用プール
    TQueue<FSortedIndexBuffer*> PendingDeletes; // 遅延削除キュー
    TQueue<uint32> FreeSlots;                   // 空きスロット管理
    uint32 FrameIndex = 0;
};

struct FSortedTriangleData
{
    const FIndexBuffer* SourceIndexBuffer; // 元インデックスバッファ
    FSortedIndexBuffer* SortedIndexBuffer; // ソート済みインデックスバッファ
    uint32 SortedFirstIndex;
    uint32 SourceFirstIndex;
    uint32 NumPrimitives;
    uint32 NumIndices;
    EPrimitiveType SourcePrimitiveType;
    EPrimitiveType SortedPrimitiveType;
};

class FSortedIndexBuffer : public FIndexBuffer
{
    static const uint32 SliceCount = 256u; // 最大スライス数
    // UAV + ShaderResource フラグで作成
    // ソート CS で RWBuffer として書き込み
    // 描画時は IndexBuffer として読み取り
};
```

---

## OIT フロー（RenderTranslucency 内）

```
RenderTranslucency()
  │
  ├─ [A] OIT::CreateOITData()
  │       → SampleDataTexture / SampleCountTexture を RDG で確保
  │       → FOITBasePassUniformParameters へバインド
  │
  ├─ [B] OIT::AddSortTrianglesPass()（SortedTriangles 有効時）
  │       → CSで全メッシュのトライアングルをソート
  │
  ├─ [C] RenderTranslucencyInner()
  │       → 各半透明メッシュの描画（通常と同じフロー）
  │       → SortedPixels 有効時はピクセルシェーダー内で
  │         OutOITSampleData への書き込みに切り替わる
  │
  └─ [D] OIT::AddOITComposePass()
          → FOITPixelCombineCS (8×8 CS)
          → SampleData を深度ソートして Alpha ブレンド
          → SceneColor に最終色を書き込み
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.OIT.SortedTriangles` | 1 | トライアングルソート有効（ReadOnly）|
| `r.OIT.SortedPixels` | 0 | MLAB 有効（プロジェクト設定・ReadOnly）|
| `r.OIT.SortedPixels.Enable` | 1 | MLAB ランタイム有効フラグ |
| `r.OIT.SortedPixels.PassType` | 3 | 適用パス（1=Standard, 2=Separate, 3=両方）|
| `r.OIT.SortedPixels.MaxSampleCount` | 4 | ピクセルあたり最大サンプル数 |
| `r.OIT.SortedPixels.Method` | 1 | 0=通常ブレンド, 1=MLAB |
| `r.OIT.SortedPixels.TransmittanceThreshold` | 0.05 | 早期終了閾値 |

---

## 関連リファレンス

- [[ref_translucent_rendering]] — `FTranslucencyPassResources` / `FTranslucentPrimSet`
- [[a_translucent_rendering]] — RenderTranslucency() メインフロー
