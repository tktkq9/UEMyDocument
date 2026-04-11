# REF: Debug / Editor 系ファイル群

- 対象ファイル（9本）:
  - `PostProcessBufferInspector.h/.cpp`
  - `PostProcessCompositeDebugPrimitives.h/.cpp`
  - `PostProcessCompositeEditorPrimitives.h/.cpp`
  - `PostProcessCompositePrimitivesCommon.h/.cpp`
  - `PostProcessGBufferHints.h/.cpp`
  - `PostProcessSelectionOutline.h/.cpp`
  - `PostProcessTestImage.h/.cpp`
  - `PostProcessStreamingAccuracyLegend.h/.cpp`
  - `DebugAlphaChannel.h/.cpp`
- 関連Details: [[g_pp_misc]]

---

## 概要

エディタやデバッグビューでのみ使用されるポストプロセスパス群。  
ゲームビルドではコンパイルされない（`WITH_EDITOR` / `WITH_DEBUG_VIEW_MODES` ガード）。

---

## PostProcessBufferInspector.h/.cpp

**特定ピクセルのシェーダーデータをリードバック**するデバッグツール。  
エディタの「Material Stats」や「GPU Visualizer」が使用する。

```cpp
// バッファインスペクターパス（ピクセル単位のデータ抽出）
FScreenPassTexture AddBufferInspectorPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input,
    const FBufferInspectorPixelData& PixelData);

struct FBufferInspectorPixelData
{
    FIntPoint PixelCoordinate;  // 検査するピクセル座標
    // 出力: リードバックバッファを通じてCPUに戻す
};
```

---

## PostProcessCompositeDebugPrimitives.h/.cpp

**デバッグプリミティブ（DrawDebugLine 等）をシーンに合成**する。  
ゲームコード側から `DrawDebugXxx` で描画したプリミティブを PP で合成する。

```cpp
FScreenPassTexture AddDebugPrimitiveCompositePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    const FMinimalSceneTextures& SceneTextures);
```

---

## PostProcessCompositeEditorPrimitives.h/.cpp

**エディタプリミティブ（ワイヤーフレーム / ギズモ / スプライン等）をシーンに合成**する。  
`WITH_EDITOR` でのみ有効。

```cpp
bool IsEditorPrimitivePassRequired(const FViewInfo& View);

FScreenPassTexture AddEditorPrimitivePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    const FMinimalSceneTextures& SceneTextures,
    FInstanceCullingManager& InstanceCullingManager);
// エディタ独自のメッシュ（ギズモ等）を深度比較付きで合成
```

---

## PostProcessCompositePrimitivesCommon.h/.cpp

デバッグ / エディタプリミティブ合成の**共通処理ユーティリティ**。

```cpp
// カスタム深度バッファを使ったプリミティブ合成の共通ロジック
class FCompositePrimitivesContext
{
    FRDGTextureRef DepthStencilTexture; // 合成判定用深度
    FRDGTextureRef PrimitiveColor;      // プリミティブカラー
    float DepthBias;                   // 深度バイアス
};
```

---

## PostProcessGBufferHints.h/.cpp

**GBuffer の潜在的な問題（法線の異常・ラフネスが 0 等）を色分けして警告表示**する。  
ビジュアルデバッグ用。

```cpp
FScreenPassTexture AddGBufferHintsPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    const FMinimalSceneTextures& SceneTextures);
```

---

## PostProcessSelectionOutline.h/.cpp

**エディタでオブジェクトを選択したときのアウトライン**を描画する。  
カスタム深度（`r.CustomDepth`）を利用してアウトラインを生成。

```cpp
bool IsEditorSelectionOutlineEnabled(const FViewInfo& View);

FScreenPassTexture AddSelectionOutlinePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    const FMinimalSceneTextures& SceneTextures);
// 選択オブジェクトのシルエットを CustomDepth から抽出しアウトライン描画

struct FSelectionOutlineParameters
{
    FLinearColor SelectionHighlightColor;      // 選択ハイライト色
    FLinearColor SubduedSelectionOutlineColor; // サブ選択色
    float SelectionHighlightIntensity;
};
```

---

## PostProcessTestImage.h/.cpp

**テストパターン画像を生成する**デバッグパス（カラーグリッド / ゾーンプレート等）。  
ディスプレイキャリブレーション・モニタテスト用。

```cpp
FScreenPassTexture AddTestImagePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input);
```

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.TestImage` | 0 | テスト画像表示（0=無効, 1=ゾーンプレート, 2=カラー）|

---

## PostProcessStreamingAccuracyLegend.h/.cpp

**テクスチャストリーミング精度の凡例**を画面に表示する。  
`Show → Texture Streaming Accuracy` と連動。

```cpp
FScreenPassTexture AddStreamingAccuracyLegendPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input);
```

---

## DebugAlphaChannel.h/.cpp

**アルファチャンネルの内容を可視化**する（アルファが正しく伝搬されているか確認）。

```cpp
FScreenPassTexture AddDebugAlphaChannelPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input);
```

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.DebugAlphaChannel` | 0 | アルファチャンネルデバッグ表示 |

---

> [!note]- WITH_EDITOR / WITH_DEBUG_VIEW_MODES によるコンパイルガード
> このファイル群のほぼすべては `#if WITH_EDITOR` または `#if WITH_DEBUG_VIEW_MODES` で囲まれており、シッピングビルドには含まれない。  
> `WITH_DEBUG_VIEW_MODES` はエディタ以外でもデバッグ可視化を有効にしたい開発ビルド向けに用意されており、`Target.cs` で `bWithDebugViewModeShaders = true` を指定することで有効化できる。  
> ゲームビルドでこれらのパスを呼ぶと空のスタブが返るか、コンパイルエラーになるため、呼び出し元でも同じガードを入れる必要がある。

> [!note]- SelectionOutline の CustomDepth 利用とステンシルマスク
> `AddSelectionOutlinePass()` (`PostProcessSelectionOutline.cpp`) はエディタで選択されたメッシュを `CustomDepth` / `CustomStencil` バッファに書き込んだ後、そのシルエットを膨張させてアウトラインを描く。  
> 選択されたオブジェクトの `bRenderCustomDepth = true` がフレームごとにセットされ、深度比較でシルエット境界を検出する。`FSelectionOutlineParameters::SelectionHighlightColor` でアウトライン色を変更できる。  
> ゲームビルドでは `IsEditorSelectionOutlineEnabled()` が常に `false` を返すためパス自体が追加されない。

> [!note]- BufferInspector の GPU リードバックとフレーム遅延
> `AddBufferInspectorPass()` は指定ピクセル座標のシェーダーデータを GPU から CPU にリードバックする。  
> RDG パスは非同期実行のため結果は **1 フレーム遅延**して CPU 側に届く。エディタの「Material Stats」ウィンドウはこの遅延を前提に更新される。  
> リードバック先は `FRDGBuffer` 経由のステージングバッファで、`RHILockBuffer` で CPU アドレスを取得して `FBufferInspectorPixelData` に書き戻す仕組み。
