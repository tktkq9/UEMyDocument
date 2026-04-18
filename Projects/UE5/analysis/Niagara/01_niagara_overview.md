# Niagara VFX システム全体概要

- 取得対象: `Engine/Plugins/FX/Niagara/Source/`
- 上位: [[UE5 解析インデックス]]

---

## Niagara の構成

| 概念 | クラス | 説明 |
|------|--------|------|
| System | `UNiagaraSystem` | 複数エミッターをまとめる最上位 |
| Emitter | `UNiagaraEmitter` | パーティクル生成単位 |
| Script | `UNiagaraScript` | HLSL ベースのノードグラフ（VM/GPU）|
| DataInterface | `UNiagaraDataInterface` | 外部データ参照（Mesh/Texture/Skeleton 等）|
| Renderer | `UNiagaraSpriteRenderer` 等 | パーティクルの描画方法 |
| Simulation Stage | `UNiagaraSimulationStageBase` | GPU 汎用計算ステージ |
| Component | `UNiagaraComponent` | ワールドへの配置 |

---

## スクリプト評価フロー

```
UNiagaraComponent::TickComponent()
  └─ FNiagaraSystemInstance::Tick()
      ├─ [CPU] FNiagaraEmitterInstance::Tick()
      │   ├─ SpawnScript  → パーティクル生成
      │   ├─ UpdateScript → 毎フレーム更新
      │   └─ EventScript  → イベント応答
      └─ [GPU] FNiagaraGPUSystemTick
          ├─ SimulationStage × N → Compute Shader
          └─ Renderer → Draw Call
```

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `NiagaraSystem.h/.cpp` | システム管理 |
| `NiagaraEmitterInstance.h/.cpp` | エミッター実行 |
| `NiagaraScript.h/.cpp` | スクリプト定義 |
| `NiagaraGPUSystemTick.h/.cpp` | GPU Tick |
| `NiagaraDataInterface.h/.cpp` | データインターフェイス基底 |
| `NiagaraRenderer.h/.cpp` | レンダラー基底 |

---

## サブシステムドキュメント

| ドキュメント | 内容 |
|------------|------|
| `Details/a_emitter_script.md` | EmitterGraph・Spawn/Update/Event スクリプト |
| `Details/b_gpu_simulation.md` | GPU シミュレーション・Simulation Stage |
| `Details/c_data_interface.md` | DataInterface（Mesh/Texture/Skeleton/Custom）|
| `Details/d_renderer.md` | Sprite/Mesh/Ribbon/Light レンダラー |
