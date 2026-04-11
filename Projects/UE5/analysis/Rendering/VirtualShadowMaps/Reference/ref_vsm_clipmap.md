# REF: VirtualShadowMapClipmap.h / .cpp

- 対象ファイル: `Private/VirtualShadowMaps/VirtualShadowMapClipmap.h` / `.cpp`
- 関連Details: [[c_vsm_clipmap]]

---

## クラス・構造体

### `FVirtualShadowMapClipmapConfig`

クリップマップの設定値をまとめる構造体。

```cpp
struct FVirtualShadowMapClipmapConfig
{
    int32 FirstLevel;               // 最初のレベル（デフォルト: 8）
    int32 LastLevel;                // 最後のレベル（デフォルト: 18）
    int32 FirstCoarseLevel;         // 粗いレベル開始（-1 = 無効）
    int32 LastCoarseLevel;          // 粗いレベル終了
    uint32 ShadowTypeId;            // シャドウ種別タグ（キャッシュキー用）
    float ResolutionLodBias;        // 解像度LODバイアス（正=低解像度）
    float ResolutionLodBiasMoving;  // ライト移動時のバイアス
    bool  bForceInvalidate;         // 強制無効化フラグ
    bool  bIsFirstPersonShadow;     // 一人称視点シャドウ
    bool  bCullDynamicTightly;      // 動的ジオメトリの厳密カリング
    bool  bUseReceiverMask;         // レシーバーマスク使用

    // CVar値からグローバル設定を取得
    static FVirtualShadowMapClipmapConfig GetGlobal();
};
```

---

### `FVirtualShadowMapClipmap`

**`FRefCountedObject` 継承**。ディレクショナルライト1つに対応するクリップマップ管理クラス。

#### コンストラクタ

```cpp
FVirtualShadowMapClipmap(
    FVirtualShadowMapArray& VirtualShadowMapArray,
    const FLightSceneInfo& InLightSceneInfo,
    const FViewMatrices& CameraViewMatrices,
    FIntPoint CameraViewRectSize,
    const FViewInfo* DependentView,
    float LightMobilityFactor,
    const FVirtualShadowMapClipmapConfig& Config)
```

`LightMobilityFactor`: ライトが移動していると大きくなり、解像度バイアスに影響。

#### 主要フィールド

| フィールド | 型 | 説明 |
|-----------|---|------|
| `LightSceneInfo` | `const FLightSceneInfo&` | 対応するライトシーン情報 |
| `DependentView` | `const FViewInfo*` | 依存するビュー（ページ要求判定用） |
| `WorldOrigin` | `FVector` | クリップマップのワールド原点 |
| `LightDirection` | `FVector` | 光源方向 |
| `WorldToLightViewRotationMatrix` | `FMatrix` | ワールド→光源ビュー回転行列 |
| `FirstLevel` | `int32` | 最初のレベル番号 |
| `ResolutionLodBias` | `float` | 解像度LODバイアス |
| `LevelData` | `TArray<FLevelData>` | 各レベルのデータ |
| `CacheEntry` | `TSharedPtr<FVirtualShadowMapPerLightCacheEntry>` | キャッシュエントリ |

#### `FLevelData`（各レベルのデータ）

```cpp
struct FLevelData
{
    int32        VirtualShadowMapId;  // このレベルのVSM ID
    FVector2D    CornerRelativeOffset; // クリップマップ原点のオフセット
    FViewMatrices ViewMatrices;       // このレベルのビュー行列
};
```

#### 主要メソッド

| メソッド | 説明 |
|---------|------|
| `GetViewMatrices(int32 ClipmapIndex) const` | 指定レベルのビュー行列取得 |
| `GetVirtualShadowMapId(int32 ClipmapIndex = 0) const` | VSM ID取得 |
| `GetLevelCount() const` | レベル数取得（= LastLevel - FirstLevel + 1） |
| `GetClipmapLevel(int32 ClipmapIndex) const` | 絶対レベル番号取得 |
| `GetPreViewTranslation(int32 ClipmapIndex) const` | プリビュー変換取得 |
| `GetViewToClipMatrix(int32 ClipmapIndex) const` | ビュー→クリップ行列取得 |
| `GetWorldToLightViewRotationMatrix() const` | ワールド→光源回転行列取得 |
| `GetLightSceneInfo() const` | ライトシーン情報取得 |
| `GetProjectionShaderData(int32 ClipmapIndex) const` | 投影シェーダーデータ取得 |
| `GetWorldOrigin() const` | クリップマップ原点取得 |
| `GetMaxRadius() const` | 最大レベルのワールド半径 |
| `GetBoundingSphere() const` | バウンディングスフィア取得 |
| `GetViewFrustumBounds(int32 ClipmapIndex) const` | 視錐台AABB取得 |
| `GetDependentView() const` | 依存ビュー取得 |
| `GetDynamicDepthCullRange(int32 ClipmapIndex) const` | 動的深度カリング範囲 |
| `OnPrimitiveRendered(const FPrimitiveSceneInfo*)` | プリミティブ描画を記録 |
| `UpdateCachedFrameData()` | フレームデータをキャッシュエントリに保存 |
| `GetCacheEntry() const` | キャッシュエントリ参照 |
| `IsFirstPersonShadow() const` | 一人称シャドウか |

#### 静的メソッド

| メソッド | 説明 |
|---------|------|
| `static float GetLevelRadius(float AbsoluteLevel)` | レベルのワールド空間半径計算（指数スケール） |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Shadow.Virtual.Clipmap.FirstLevel` | 8 | 最初のクリップマップレベル |
| `r.Shadow.Virtual.Clipmap.LastLevel` | 18 | 最後のクリップマップレベル |
| `r.Shadow.Virtual.Clipmap.FirstCoarseLevel` | -1 | 粗いレベル開始（-1=無効） |
| `r.Shadow.Virtual.Clipmap.LastCoarseLevel` | -1 | 粗いレベル終了 |
| `r.Shadow.Virtual.Clipmap.ZRangeScale` | 1.0 | 深度範囲スケール |
| `r.Shadow.Virtual.Clipmap.MinCameraViewportWidth` | 1 | 最小カメラビューポート幅 |
| `r.Shadow.Virtual.Clipmap.WPODisableDistance` | 0 | World Position Offset無効距離 |
| `r.Shadow.Virtual.Clipmap.WPODisableDistance.LodBias` | 0 | WPO無効LODバイアス |
| `r.Shadow.Virtual.Clipmap.WPODisableDistance.InvalidateOnScaleChange` | 0 | スケール変更時に無効化 |
| `r.Shadow.Virtual.Clipmap.CullDynamicTightly` | 0 | 動的ジオメトリ厳密カリング |
| `r.Shadow.Virtual.ResolutionLodBiasDirectional` | 0.0 | ディレクショナル解像度LODバイアス |
| `r.Shadow.Virtual.ResolutionLodBiasDirectionalMoving` | 0.0 | ライト移動時LODバイアス |
| `r.Shadow.Virtual.MarkCoarsePagesDirectional` | 1 | 粗ページマーキング有効 |
| `r.Shadow.Virtual.UseReceiverMaskDirectional` | 0 | レシーバーマスク使用 |
| `r.Shadow.Virtual.Cache.ForceInvalidateDirectional` | 0 | 強制無効化（デバッグ） |
| `r.Shadow.Virtual.SMRT.RayCountDirectional` | 7 | ディレクショナルSMRTレイ数 |
| `r.Shadow.Virtual.SMRT.SamplesPerRayDirectional` | 8 | レイあたりサンプル数 |
| `r.Shadow.Virtual.SMRT.ExtrapolateMaxSlopeDirectional` | 5.0 | 傾き外挿上限 |
| `r.Shadow.Virtual.SMRT.RayLengthScaleDirectional` | 1.5 | レイ長スケール |
| `r.Shadow.Virtual.SMRT.TexelDitherScaleDirectional` | 2.0 | テクセルディザースケール |
| `r.FirstPerson.Shadow.Virtual.Clipmap.PixelRequestBias` | 2.0 | 一人称ページ要求バイアス |
| `r.FirstPerson.Shadow.Virtual.Clipmap.RequestMinLevelClamp` | 8 | 一人称最小レベルクランプ |

---

> [!note]- GetLevelRadius の指数スケール
> `FVirtualShadowMapClipmap::GetLevelRadius(float AbsoluteLevel)` は `2^AbsoluteLevel` スケールでワールド空間半径を計算する。  
> Level 8 は近距離（約 256m）、Level 18 は遠距離（約 262km）をカバーする。  
> 1レベル上がるごとに半径が2倍になり、仮想ページの解像度は半分になる（ページ数は同じ）。  
> これにより近傍は高解像度、遠距離は自動的に低解像度となる連続的なシャドウ品質が実現する。

> [!note]- CornerRelativeOffset とクリップマップの移動
> `FLevelData::CornerRelativeOffset` はクリップマップの仮想アドレス空間内の原点オフセット。  
> カメラが移動するとオフセットが更新されるが、ページ単位でのスナップ（整数ページ移動のみ）により  
> 大部分のページが再利用でき、境界部分のみ再描画する仕組みになっている。  
> シェーダー内で `ClipmapCornerRelativeOffset` としてサンプリング座標の補正に使われる。

> [!note]- bIsFirstPersonShadow の専用クリップマップ
> `bIsFirstPersonShadow = true` の場合、通常シーン用クリップマップとは別に専用クリップマップが生成される。  
> 一人称視点のウェポン・ハンドはカメラ極近傍に存在し、通常クリップマップの最近傍レベルでも解像度不足になるため、  
> 専用の近距離クリップマップが必要。ページ要求バイアスを `r.FirstPerson.Shadow.Virtual.Clipmap.PixelRequestBias` で調整できる。
