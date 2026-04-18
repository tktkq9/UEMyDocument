# Audio システム全体概要

- 取得対象: `Engine/Source/Runtime/AudioMixer/`, `Engine/Plugins/Runtime/MetasoundEngine/`
- 上位: [[UE5 解析インデックス]]

---

## UE5 Audio システムの構成

UE5 の Audio は **AudioMixer** ベースで動作し、  
**MetaSound** がプロシージャルサウンドの新標準となっている。

| 概念 | クラス | 説明 |
|------|--------|------|
| サウンドウェーブ | `USoundWave` | PCM サンプルデータ |
| SoundCue | `USoundCue` | ノードベースのサウンドグラフ（旧来）|
| MetaSound | `UMetaSoundSource` | DSP グラフベースの新サウンドシステム |
| AudioComponent | `UAudioComponent` | ワールド内でのサウンド再生 |
| Attenuation | `USoundAttenuation` | 距離減衰・スペーシャライズ |
| Submix | `USoundSubmix` | ミキサー階層・エフェクトチェーン |
| Concurrency | `USoundConcurrency` | 同時再生数制限 |
| Quartz | `UQuartzSubsystem` | 音楽クロック同期 |

---

## 音声処理フロー

```
UAudioComponent::Play()
  └─ FAudioDevice::PlaySoundAtLocation()
      └─ FActiveSound（再生管理）
          ├─ USoundAttenuation → 距離・方向計算
          ├─ UAudioMixer → バス・Submix へルーティング
          └─ FAudioMixerBuffer → PCM デコード
              └─ ISpatializationPlugin → 空間音声
                  └─ FAudioMixerSubmix → DSP チェーン
                      └─ OutputBuffer → デバイス出力
```

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `AudioMixerDevice.h/.cpp` | AudioMixer のコアデバイス |
| `AudioMixerSubmix.h/.cpp` | Submix 処理 |
| `AudioComponent.h/.cpp` | ワールド内再生コンポーネント |
| `MetaSoundSource.h/.cpp` | MetaSound ソース |
| `MetasoundGenerator.h/.cpp` | MetaSound DSP グラフ評価 |
| `SoundAttenuation.h/.cpp` | 減衰設定 |
| `QuartzSubsystem.h/.cpp` | 音楽クロック |

---

## サブシステムドキュメント

| ドキュメント | 内容 |
|------------|------|
| `Details/a_audio_mixer.md` | AudioMixer アーキテクチャ・バッファ処理 |
| `Details/b_metasound.md` | MetaSound DSP グラフ・ノード・カスタムノード |
| `Details/c_attenuation_spatialization.md` | 距離減衰・HRTF・スペーシャライズ |
| `Details/d_submix.md` | Submix 階層・エフェクトチェーン・EQ |
| `Details/e_quartz.md` | Quartz クロック・ビート同期・レイテンシ |
