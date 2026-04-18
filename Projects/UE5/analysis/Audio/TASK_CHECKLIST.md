# Audio ドキュメント チェックリスト

## 概要
- [ ] `01_audio_overview.md` … Audio 全体概要（更新予定）
- [ ] `_source_map.md` … ソースマップ

---

## サブフォルダ構成

### AudioMixer — ミキサーエンジン
- [ ] `AudioMixer/01_overview.md` … AudioMixer 概要
- [ ] `AudioMixer/Details/a_mixer_device.md` … FAudioMixerDevice・プラットフォーム初期化・バッファ処理
- [ ] `AudioMixer/Details/b_source_manager.md` … FAudioMixerSourceManager・ソースボイス管理・プール
- [ ] `AudioMixer/Details/c_thread_model.md` … AudioThread・RenderThread 連携・コマンドキュー
- [ ] `AudioMixer/Reference/ref_mixer_api.md` … FAudioDevice / FAudioMixerDevice API

### MetaSound — プロシージャルオーディオ
- [ ] `MetaSound/01_overview.md` … MetaSound 概要
- [ ] `MetaSound/Details/a_metasound_graph.md` … FMetasoundFrontend・グラフ構造・ノード評価順序
- [ ] `MetaSound/Details/b_metasound_nodes.md` … 標準ノード（Oscillator/Filter/Envelope/Trigger）
- [ ] `MetaSound/Details/c_custom_node.md` … カスタム MetaSound ノード実装・FNodeFacade
- [ ] `MetaSound/Reference/ref_metasound_api.md` … UMetaSoundSource / FMetasoundFrontend API
- [ ] `MetaSound/Reference/ref_metasound_nodes.md` … 標準 MetaSound ノード全一覧

### Attenuation — 減衰・空間化
- [ ] `Attenuation/01_overview.md` … Attenuation 概要
- [ ] `Attenuation/Details/a_attenuation.md` … FSoundAttenuationSettings・距離カーブ・FalloffMode
- [ ] `Attenuation/Details/b_spatialization.md` … HRTF・Binaural・ISpatializationPlugin・Panning
- [ ] `Attenuation/Details/c_reverb.md` … AudioVolume・ReverbEffect・SubmixSend による残響
- [ ] `Attenuation/Reference/ref_attenuation_api.md` … FSoundAttenuationSettings / UAudioComponent API

### Submix — サブミックス・エフェクト
- [ ] `Submix/01_overview.md` … Submix 概要
- [ ] `Submix/Details/a_submix_graph.md` … USoundSubmixBase・サブミックス階層・ルーティング
- [ ] `Submix/Details/b_submix_effects.md` … USoundEffectSubmixPreset・EQ/Compressor/Reverb チェーン
- [ ] `Submix/Details/c_submix_recording.md` … サブミックス録音・StartRecording/FinishRecording
- [ ] `Submix/Reference/ref_submix_api.md` … USoundSubmixBase / USoundEffectSubmixPreset API

### Quartz — 定量音楽システム
- [ ] `Quartz/01_overview.md` … Quartz 概要
- [ ] `Quartz/Details/a_quartz_clock.md` … UQuartzClockHandle・BPM・拍子設定・Transport
- [ ] `Quartz/Details/b_quantized_events.md` … FOnQuartzCommandEvent・QuantizedPlaySound・サブスクライブ
- [ ] `Quartz/Reference/ref_quartz_api.md` … UQuartzClockHandle / UQuartzSubsystem API

---

## 進捗サマリ

| サブフォルダ | 概要 | Details | Reference | 完了 |
|------------|------|---------|-----------|------|
| AudioMixer | 0/1 | 0/3 | 0/1 | 0/5 |
| MetaSound | 0/1 | 0/3 | 0/2 | 0/6 |
| Attenuation | 0/1 | 0/3 | 0/1 | 0/5 |
| Submix | 0/1 | 0/3 | 0/1 | 0/5 |
| Quartz | 0/1 | 0/2 | 0/1 | 0/4 |

**合計**: 概要 1 + ソースマップ 1 + サブフォルダ概要 5 + Details 14 + Reference 6 = **27 ファイル**
