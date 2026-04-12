# REF: VSM Projection シェーダー

- グループ: d - Projection
- 詳細: [[detail_projection]]
- CPU リファレンス: [[ref_vsm_projection]] | [[ref_vsm_shaders]]
- ソース: `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapProjection.usf`
          `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapProjectionComposite.usf`

---

## VirtualShadowMapProjection.usf

### VirtualShadowMapProjection

```hlsl
[numthreads(WORK_TILE_SIZE, WORK_TILE_SIZE, 1)]
void VirtualShadowMapProjection(
    uint3 GroupId          : SV_GroupID,
    uint  GroupIndex       : SV_GroupIndex,
    uint3 DispatchThreadId : SV_DispatchThreadID
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | SceneDepth を元に TranslatedWorldPosition を復元し、SMRT でシャドウ判定を実行して ShadowFactor または ShadowMaskBits に書き込む |
| **スレッドグループ** | `WORK_TILE_SIZE` × `WORK_TILE_SIZE`（Morton Order でピクセルに割り当て）|
| **入力** | `SceneTexturesStruct.SceneDepthTexture`, `VirtualShadowMap.PageTable`, `PhysicalPagePool`, `BlueNoise` |
| **出力** | `OutShadowFactor`（float2, Per-Light 時）または `OutShadowMaskBits`（uint4, One-Pass 時）|
| **CPU 関数** | `RenderVirtualShadowMapProjection()` |

#### パーミュテーション

| マクロ | 値 | 説明 |
|--------|-----|------|
| `ONE_PASS_PROJECTION` | 0/1 | 全ローカルライトを1 Dispatch で処理（0=Per-Light）|
| `DIRECTIONAL_LIGHT` | 0/1 | ディレクショナルライト専用処理（クリップマップ参照）|
| `INPUT_TYPE` | 0/1 | 0=GBuffer, 1=HairStrands |
| `USE_TILE_LIST` | 0/1 | タイルリストを使った Indirect Dispatch |
| `VISUALIZE_OUTPUT` | 0/1 | デバッグビジュアライゼーション出力付き |

#### 主要内部関数

| 関数 | ヘッダー | 役割 |
|-----|---------|------|
| `ProjectLight()` | `VirtualShadowMapProjection.usf` | メインのシャドウサンプリング関数 |
| `VirtualShadowMapGetLightsGridHeader()` | `VirtualShadowMapLightGrid.ush` | One-Pass 用ライトグリッドヘッダー取得 |
| `FilterVirtualShadowMapSampleResult()` | `VirtualShadowMapProjectionFilter.ush` | サンプル結果のフィルタリング |
| `PackShadowMask()` | `VirtualShadowMapMaskBitsCommon.ush` | One-Pass 結果のパック |
| `GetProjectionShadingInfo()` | `VirtualShadowMapProjectionStructs.ush` | ピクセルのシェーディング情報取得 |

---

## VirtualShadowMapProjectionComposite.usf

### VirtualShadowMapCompositeTileVS

```hlsl
void VirtualShadowMapCompositeTileVS(
    in uint InstanceId  : SV_InstanceID,
    in uint VertexId    : SV_VertexID,
    out float4 Position : SV_POSITION
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | タイルリストのタイル座標からクリップ空間のクワッド頂点を生成（タイルベース描画）|
| **入力** | `TileListData`（タイル座標パック uint）|
| **出力** | `SV_POSITION`（タイル範囲のクリップ空間矩形）|

### VirtualShadowMapCompositePS

```hlsl
void VirtualShadowMapCompositePS(
    in float4 SvPosition       : SV_Position,
    out float4 OutShadowMask   : SV_Target
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | `InputShadowFactor`（float2）を Shadow Attenuation フォーマットにエンコードして出力 |
| **入力** | `InputShadowFactor`（Per-Light Projection の出力）|
| **出力** | `OutShadowMask`（float4: `EncodeLightAttenuation` エンコード済み）|
| **bModulateRGB** | 1 のとき: ShadowFactor.x を RGBA に直接出力（MegaLights 統合用）|
| **CPU 関数** | `CompositeVirtualShadowMapMask()` |

### VirtualShadowMapCompositeFromMaskBitsPS

```hlsl
void VirtualShadowMapCompositeFromMaskBitsPS(
    in float4 SvPosition       : SV_Position,
    out float4 OutShadowMask   : SV_Target
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | One-Pass Projection で生成した `ShadowMaskBits`（uint4）から指定ライトの Shadow Factor を抽出して出力 |
| **入力** | `ShadowMaskBits`（uint4 packed）, `CompositeVirtualShadowMapId`（ライトインデックス）|
| **出力** | `OutShadowMask`（float4, Shadow Attenuation）|

---

## 主要シェーダーパラメーター

| パラメーター | 型 | 内容 |
|------------|-----|------|
| `VirtualShadowMap` | `FVirtualShadowMapUniformParameters` | ページテーブル / 物理ページプール / 全 VSM 設定 |
| `ProjectionRect` | `int4` | 処理するピクセル矩形 |
| `LightUniformVirtualShadowMapId` | `int` | Per-Light モードの VSM ID |
| `SubsurfaceMinSourceRadius` | `float` | サブサーフェスマテリアルの最小ライト半径 |
| `ScreenRayLength` | `float` | SMRT スクリーンレイの最大長 |
| `InputShadowFactor` | `Texture2D<float2>` | Composite 用 Shadow Factor テクスチャ |
| `ShadowMaskBits` | `Texture2D<uint4>` | One-Pass 用 Shadow Mask テクスチャ |
| `CompositeVirtualShadowMapId` | `int` | Composite 時のターゲット VSM ID |
| `bModulateRGB` | `uint` | RGB チャンネルに ShadowFactor を直接書き込むか |

---

## 共有ヘッダー

| ファイル | 役割 |
|---------|------|
| `VirtualShadowMapSMRTCommon.ush` | SMRT 基本構造体 / 定数定義 |
| `VirtualShadowMapSMRTTemplate.ush` | `ExtrapolateMaxSlope()` / SMRT サンプリングループ |
| `VirtualShadowMapProjectionCommon.ush` | `SampleVirtualShadowMap()` / ページ参照共通ロジック |
| `VirtualShadowMapProjectionDirectional.ush` | ディレクショナルライト専用クリップマップ参照 |
| `VirtualShadowMapProjectionSpot.ush` | スポット/ポイントライト専用投影変換 |
| `VirtualShadowMapScreenRayTrace.ush` | スクリーンスペースレイトレースのユーティリティ |
| `VirtualShadowMapTransmissionCommon.ush` | 半透明マテリアルのシャドウ透過計算 |
| `VirtualShadowMapMaskBitsCommon.ush` | One-Pass 用 ShadowMask のパック/アンパック |
| `VirtualShadowMapProjectionFilter.ush` | サンプル結果のフィルタ処理 |

---

> [!note]- SMRT（Shadow Map Ray Tracing）の仕組み
> `ProjectLight()` 内の SMRT は通常の PCF とは異なり、ライト方向にサンプル点を複数取って  
> 「どの距離から遮蔽が始まるか」を推定し、そこから半影幅を計算する。  
> `ExtrapolateMaxSlope()` が深度差の最大傾斜を外挿して遮蔽物の距離を推定し、  
> これを元にソフトシャドウの強度を決定する。Blue Noise ジッターとテンポラル安定化を組み合わせて  
> 少ないサンプル数でも品質の高いソフトシャドウを実現する。

> [!note]- One-Pass Projection の制限と利点
> `ONE_PASS_PROJECTION=1` は1 Dispatch で最大 `GetPackedShadowMaskMaxLightCount()` ライト分の  
> シャドウを `uint4`（ShadowMaskBits）にパックする。これによりローカルライトのシャドウを  
> 1パスで処理できるが、ライト数が上限を超えると `VSM_STAT_OVERFLOW_FLAG` が立つ。  
> MegaLights はソート末尾に配置され、先頭ライトにヒットした段階でループを打ち切る。

> [!note]- VSM Projection と従来シャドウマップの違い
> 従来のシャドウマップはライトの Projection でシャドウ解像度が固定だが、  
> VSM は仮想ページテーブルのページサイズがカメラからの距離に応じて自動調整される。  
> ページが物理アトラス上に散在するため、`SampleVirtualShadowMap()` は  
> 仮想→物理ページテーブルのルックアップを経てから深度サンプリングを行う。
