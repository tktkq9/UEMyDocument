# REF: TemporalSuperResolution.cpp

- 対象ファイル: `Private/PostProcess/TemporalSuperResolution.cpp`（ヘッダなし）
- 関連Details: [[b_pp_taa_tsr]]

---

## 概要

TSR（Temporal Super Resolution）は UE5 標準のテンポラルアップスケーラー。  
`ITemporalUpscaler` インターフェースを実装し、低解像度入力をフル解像度に引き上げながら AA を行う。  
実装は単一 .cpp に集約されており、専用ヘッダは持たない（`TemporalAA.h` 内の `ITemporalUpscaler` を使用）。

---

## ITemporalUpscaler インターフェース（TemporalAA.h で定義）

```cpp
class ITemporalUpscaler
{
public:
    struct FPassInputs
    {
        FRDGTextureRef SceneColorTexture;     // 低解像度 SceneColor
        FRDGTextureRef SceneDepthTexture;     // シーン深度（フル解像度）
        FRDGTextureRef SceneVelocityTexture;  // ベロシティ（フル解像度）
        FRDGTextureRef EyeAdaptationTexture;  // 露出値
        bool bAllowFullResInput;              // フル解像度入力許可
        bool bGenerateOutputMip1;            // 出力 MIP1 生成（次フレームTSR履歴用）
        float SequenceNumber;                // フレームシーケンス番号
    };

    struct FOutputs
    {
        FRDGTextureRef FullRes;   // フル解像度出力（TSR 結果）
        FRDGTextureRef HalfRes;   // ハーフ解像度出力（オプション）
        FRDGTextureRef Mip1;      // MIP1（TSR 履歴のダウンサンプル）
    };

    virtual FOutputs AddPasses(
        FRDGBuilder& GraphBuilder,
        const FViewInfo& View,
        const FPassInputs& PassInputs) const = 0;

    virtual const TCHAR* GetDebugName() const = 0;
    virtual float GetMinUpsampleResolutionFraction() const;
    virtual float GetMaxUpsampleResolutionFraction() const;
};
```

---

## TSR 内部パス構成

```
FTSRHistory（FSceneViewState に保持される履歴バッファ群）
  ├── HighFrequency         // 高周波数成分の履歴
  ├── LowFrequency          // 低周波数成分の履歴
  ├── Metadata              // メタデータ（サンプルカウント・ウェイト）
  ├── Translucency          // 半透明専用の履歴
  └── GuardBand             // ビュー外ガードバンド

TSR パス実行順:
[1] TSRClearPrevTexturesCS      … 新規ビューの履歴バッファ初期化
[2] TSRDilateVelocityCS         … ベロシティバッファの近傍最大値拡張
                                    （エッジを通過する速度のカバレッジ改善）
[3] TSRDecimateHistoryCS        … 履歴をダウンサンプルして Translucency 比較用を準備
[4] TSRCompareTranslucencyCS    … 半透明テクスチャの変化検出（ゴースト防止）
[5] TSRRejectShadingCS          … シェーディング変化に基づくゴースト除去
                                    ● 履歴の輝度クリッピング（AABB クリップ）
                                    ● 鮮明度スコア計算
                                    ● 速度ベース再投影の品質推定
[6] TSRSpatialAntiAliasingCS    … 空間的エッジ処理
                                    （ジャギーの空間方向での補完）
[7] TSRAccumulateCS             … 現フレームと履歴の混合（メインパス）
                                    ● 履歴ウェイトの計算
                                    ● 線形補間でブレンド
                                    ● フル解像度出力を生成
[8] TSRPostfilterHistoryCS      … 履歴バッファのポストフィルタリング
                                    （次フレームに持ち越す履歴の品質改善）
[9] TSRUpdateGuardBandCS        … ガードバンド更新
                                    （ビュー境界外のデータを処理）
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.TSR.History.ScreenPercentage` | 200 | 履歴バッファ解像度（入力解像度の%）|
| `r.TSR.History.SampleCount` | 16 | 蓄積サンプル数上限 |
| `r.TSR.History.UpdateSpeed` | 3 | 履歴更新速度（大きいほど速く更新）|
| `r.TSR.Velocity.WeightClampingSampleCount` | 4 | 速度ウェイトクランプ用サンプル数 |
| `r.TSR.ShadingRejection.Flickering` | 1 | フリッカリング検出・除去 |
| `r.TSR.ShadingRejection.Flickering.Period` | 3 | フリッカリング周期（フレーム数）|
| `r.TSR.AsyncCompute` | 0 | 非同期コンピュート使用 |
| `r.TSR.Translucency.PreviousFrameRejection` | 1 | 前フレーム半透明の拒否 |
| `r.TSR.16BitVALU` | 1 | 16bit VALU 使用（パフォーマンス向上）|
| `r.TSR.Output.HalfRes` | 0 | ハーフ解像度出力生成 |
| `r.TSR.Subpixel.Details` | 1 | サブピクセルディテール保持 |
| `r.TSR.Debug.ArraySize` | 0 | デバッグ配列サイズ（0=無効）|
