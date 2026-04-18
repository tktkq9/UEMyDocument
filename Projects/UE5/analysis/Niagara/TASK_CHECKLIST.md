# Niagara ドキュメント チェックリスト

## 概要
- [ ] `01_niagara_overview.md` … Niagara 全体概要（更新予定）
- [ ] `_source_map.md` … ソースマップ

---

## サブフォルダ構成

### EmitterSystem — エミッター・システム・コンポーネント
- [ ] `EmitterSystem/01_overview.md` … EmitterSystem 概要
- [ ] `EmitterSystem/Details/a_niagara_system.md` … UNiagaraSystem・FNiagaraSystemInstance・Tick モデル
- [ ] `EmitterSystem/Details/b_niagara_emitter.md` … UNiagaraEmitter・SimTarget（CPU/GPU）・EmitterHandle
- [ ] `EmitterSystem/Details/c_niagara_component.md` … UNiagaraComponent・Activation/Deactivation・パラメータ設定
- [ ] `EmitterSystem/Details/d_niagara_script.md` … UNiagaraScript・ScriptUsage（Spawn/Update/Event）・コンパイル
- [ ] `EmitterSystem/Reference/ref_system_api.md` … UNiagaraSystem / FNiagaraSystemInstance API
- [ ] `EmitterSystem/Reference/ref_component_api.md` … UNiagaraComponent API

### GPUSimulation — GPU パーティクルシミュレーション
- [ ] `GPUSimulation/01_overview.md` … GPUSimulation 概要
- [ ] `GPUSimulation/Details/a_gpu_sim_stage.md` … FNiagaraGPUSimulationStage・ComputeShader ディスパッチ
- [ ] `GPUSimulation/Details/b_particle_buffer.md` … FNiagaraDataBuffer・ParticleAttribute・GPU メモリレイアウト
- [ ] `GPUSimulation/Details/c_gpu_readback.md` … GPU→CPU リードバック・イベント・Spawn からの出力
- [ ] `GPUSimulation/Reference/ref_gpu_sim_api.md` … FNiagaraGPUInstanceCountManager / FNiagaraGpuComputeDispatch API

### DataInterface — データインターフェース
- [ ] `DataInterface/01_overview.md` … DataInterface 概要
- [ ] `DataInterface/Details/a_mesh_di.md` … UNiagaraDataInterfaceStaticMesh・頂点/三角形サンプリング
- [ ] `DataInterface/Details/b_texture_di.md` … UNiagaraDataInterfaceTexture・SampleTexture・GPU サンプラー
- [ ] `DataInterface/Details/c_skeleton_di.md` … UNiagaraDataInterfaceSkeletalMesh・ボーン位置・ソケット
- [ ] `DataInterface/Details/d_collision_di.md` … UNiagaraDataInterfaceCollisionQuery・DepthBuffer/DistanceField
- [ ] `DataInterface/Details/e_custom_di.md` … カスタム DataInterface 実装・GetFunctions/GetVMExternalFunction
- [ ] `DataInterface/Reference/ref_di_api.md` … UNiagaraDataInterface 基底 API
- [ ] `DataInterface/Reference/ref_standard_di.md` … 標準 DataInterface 全一覧

### Renderers — レンダラー
- [ ] `Renderers/01_overview.md` … Renderers 概要
- [ ] `Renderers/Details/a_sprite_renderer.md` … UNiagaraSpriteRendererProperties・SubUV・Alignment・Facing
- [ ] `Renderers/Details/b_mesh_renderer.md` … UNiagaraMeshRendererProperties・インスタンス描画・LOD
- [ ] `Renderers/Details/c_ribbon_renderer.md` … UNiagaraRibbonRendererProperties・テッセレーション・UV
- [ ] `Renderers/Details/d_light_renderer.md` … UNiagaraLightRendererProperties・動的ライト生成
- [ ] `Renderers/Reference/ref_renderer_api.md` … UNiagaraRendererProperties 派生 API

### Modules — モジュール・ノードグラフ
- [ ] `Modules/01_overview.md` … Modules 概要
- [ ] `Modules/Details/a_standard_modules.md` … 標準モジュール（InitialSize/Velocity/Forces/Lifetime）
- [ ] `Modules/Details/b_niagara_graph.md` … UNiagaraGraph・ノード評価・HLSLトランスレーション
- [ ] `Modules/Details/c_dynamic_inputs.md` … DynamicInput・Scratch パッド・インラインスクリプト
- [ ] `Modules/Reference/ref_module_list.md` … 標準 Niagara モジュール全一覧
- [ ] `Modules/Reference/ref_niagara_hlsl.md` … Niagara HLSL 関数・組み込み関数

---

## 進捗サマリ

| サブフォルダ | 概要 | Details | Reference | 完了 |
|------------|------|---------|-----------|------|
| EmitterSystem | 0/1 | 0/4 | 0/2 | 0/7 |
| GPUSimulation | 0/1 | 0/3 | 0/1 | 0/5 |
| DataInterface | 0/1 | 0/5 | 0/2 | 0/8 |
| Renderers | 0/1 | 0/4 | 0/1 | 0/6 |
| Modules | 0/1 | 0/3 | 0/2 | 0/6 |

**合計**: 概要 1 + ソースマップ 1 + サブフォルダ概要 5 + Details 19 + Reference 8 = **34 ファイル**
