# VSM c: VirtualShadowMapClipmap（ディレクショナルライト）

- 対象: `VirtualShadowMaps/VirtualShadowMapClipmap.h/.cpp`
- 上位: [[04_vsm_overview]]

---

## 役割

`FVirtualShadowMapClipmap` は**ディレクショナルライト専用の VSM 管理クラス**。  
複数の同心円状クリップマップレベルを持ち、カメラ近傍は高解像度・遠方は低解像度で  
連続的にシャドウをカバーする。

従来の CSM（Cascade Shadow Maps）の境界問題を解消し、  
仮想ページングと組み合わせて必要部分だけを描画する。

---

## クリップマップ構造

```
クリップマップレベル（デフォルト: Level 8 〜 18）

Level  8: 近距離（最高解像度）
Level  9:
  ...
Level 18: 遠距離（最低解像度）

各レベルは1つのVSM (FVirtualShadowMap) を持つ
レベルごとに視錐台サイズが2倍になり、半径も倍増

FirstCoarseLevel / LastCoarseLevel: 粗い遠距離レベル（別扱い可）
```

---

## 設定構造体

```cpp
struct FVirtualShadowMapClipmapConfig
{
    int32 FirstLevel;           // 最初のレベル (デフォルト: 8)
    int32 LastLevel;            // 最後のレベル (デフォルト: 18)
    int32 FirstCoarseLevel;     // 粗いレベルの開始 (-1=なし)
    int32 LastCoarseLevel;      // 粗いレベルの終了
    float ResolutionLodBias;    // 解像度LODバイアス（正=低解像度）
    float ResolutionLodBiasMoving; // ライト移動時のバイアス
    bool  bForceInvalidate;     // 強制無効化
    bool  bIsFirstPersonShadow; // 一人称視点シャドウ
    bool  bCullDynamicTightly;  // 動的ジオメトリの厳密カリング
    bool  bUseReceiverMask;     // レシーバーマスク使用

    static FVirtualShadowMapClipmapConfig GetGlobal(); // CVar値から取得
};
```

---

## レベル半径の計算

```cpp
// レベルのワールド空間半径
static float GetLevelRadius(float AbsoluteLevel)
// → 2^AbsoluteLevel スケールで増加（指数的拡大）

// 各レベルのビュー行列（正射影）
FViewMatrices GetViewMatrices(int32 ClipmapIndex) const
// → 光源方向に垂直な平行投影ビュー
```

---

## 動的深度カリング

```cpp
FVector2f GetDynamicDepthCullRange(int32 ClipmapIndex) const
// 動的ジオメトリのカリング深度範囲を返す
// → 静的オブジェクトがキャッシュ済みの領域を動的に除外
```

---

## キャッシュとの連携

```
FVirtualShadowMapClipmap
  └── TSharedPtr<FVirtualShadowMapPerLightCacheEntry> CacheEntry
         ├── ShadowMapEntries[LevelIndex]  ← 各レベルのキャッシュ
         └── RenderedPrimitives            ← 描画済みプリミティブ

OnPrimitiveRendered(FPrimitiveSceneInfo*)
  → RenderedPrimitives に記録（キャッシュ有効性判定に使用）

UpdateCachedFrameData()
  → 現フレームのデータをエントリに保存（次フレームで再利用）
```

---

## 投影データ（ライティングパス用）

```cpp
// シェーダーに渡す投影データ（各レベル1つ）
FVirtualShadowMapProjectionShaderData
  ├── ShadowViewToClipMatrix            // シャドウビュー→クリップ行列
  ├── TranslatedWorldToShadowUVMatrix   // ワールド→シャドウUV行列
  ├── LightDirection                    // 光源方向
  ├── ClipmapLevel_ClipmapLevelCountRemaining  // レベル番号（正=クリップマップ）
  └── ClipmapCornerRelativeOffset       // クリップマップの原点オフセット
```

`ClipmapLevel_ClipmapLevelCountRemaining` が正の値 → ディレクショナルライト（クリップマップ）  
負の値（-1）→ ローカルライト

---

## First Person Shadow（一人称視点）

`bIsFirstPersonShadow = true` の専用クリップマップを別途生成し、  
一人称視点のウェポン/ハンド等を通常シーンと独立してシャドウ処理する。

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.FirstPerson.Shadow.Virtual.Clipmap.PixelRequestBias` | 2.0 | ページ要求解像度バイアス |
| `r.FirstPerson.Shadow.Virtual.Clipmap.RequestMinLevelClamp` | 8 | 最小レベルクランプ |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Shadow.Virtual.Clipmap.FirstLevel` | 8 | 最初のクリップマップレベル |
| `r.Shadow.Virtual.Clipmap.LastLevel` | 18 | 最後のクリップマップレベル |
| `r.Shadow.Virtual.Clipmap.FirstCoarseLevel` | -1 | 粗いレベルの開始（-1=無効） |
| `r.Shadow.Virtual.Clipmap.LastCoarseLevel` | -1 | 粗いレベルの終了 |
| `r.Shadow.Virtual.Clipmap.ZRangeScale` | 1.0 | 深度範囲スケール |
| `r.Shadow.Virtual.Clipmap.WPODisableDistance` | 0 | WPO（World Position Offset）無効距離 |
| `r.Shadow.Virtual.Clipmap.CullDynamicTightly` | 0 | 動的ジオメトリの厳密カリング |
| `r.Shadow.Virtual.ResolutionLodBiasDirectional` | 0.0 | ディレクショナルライト解像度LODバイアス |
| `r.Shadow.Virtual.ResolutionLodBiasDirectionalMoving` | 0.0 | 移動時の解像度LODバイアス |
| `r.Shadow.Virtual.MarkCoarsePagesDirectional` | 1 | 粗ページマーキング |
| `r.Shadow.Virtual.Cache.ForceInvalidateDirectional` | 0 | デバッグ用強制無効化 |
| `r.Shadow.Virtual.UseReceiverMaskDirectional` | 0 | レシーバーマスク使用 |
| `r.Shadow.Virtual.SMRT.RayCountDirectional` | 7 | SMRT ディレクショナルレイ数 |
| `r.Shadow.Virtual.SMRT.SamplesPerRayDirectional` | 8 | SMRT レイあたりサンプル数 |
| `r.Shadow.Virtual.SMRT.ExtrapolateMaxSlopeDirectional` | 5.0 | SMRT 傾き外挿最大値 |
