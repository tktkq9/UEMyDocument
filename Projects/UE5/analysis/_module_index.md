# UE5 モジュールインデックス

Claude がソースコードを探索する際のナビゲーション用。  
各システムの **ソースパス** と **解析フォルダ** を集約する。

- UE5 ソースルート: `D:\UnrealEngine\`
- 解析メモルート: `D:\Learning\Projects\UE5\analysis\`
- 更新日: 2026-04-18

---

## Rendering（完了）

| モジュール | ソースパス | 解析フォルダ |
|-----------|----------|------------|
| Renderer | `Engine/Source/Runtime/Renderer/Private/` | `Rendering/` |
| RHI | `Engine/Source/Runtime/RHI/` | — |
| RenderCore | `Engine/Source/Runtime/RenderCore/` | — |
| Shaders | `Engine/Shaders/Private/` | `Rendering/GPU/` |

## GAS（Gameplay Ability System）

| モジュール | ソースパス | 解析フォルダ |
|-----------|----------|------------|
| GameplayAbilities | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/` | `GAS/` |
| GameplayTags | `Engine/Source/Runtime/GameplayTags/` | `GAS/`（タグ関連） |
| GameplayTasks | `Engine/Source/Runtime/GameplayTasks/` | `GAS/`（タスク関連） |

## Animation

| モジュール | ソースパス | 解析フォルダ |
|-----------|----------|------------|
| Engine/Animation | `Engine/Source/Runtime/Engine/Classes/Animation/` | `Animation/` |
| Engine/Animation (Private) | `Engine/Source/Runtime/Engine/Private/Animation/` | `Animation/` |
| AnimGraphRuntime | `Engine/Source/Runtime/AnimGraphRuntime/` | `Animation/` |
| AnimationCore | `Engine/Source/Runtime/AnimationCore/` | `Animation/` |

## AI

| モジュール | ソースパス | 解析フォルダ |
|-----------|----------|------------|
| AIModule | `Engine/Source/Runtime/AIModule/` | `AI/` |
| NavigationSystem | `Engine/Source/Runtime/NavigationSystem/` | `AI/` |

## Input

| モジュール | ソースパス | 解析フォルダ |
|-----------|----------|------------|
| EnhancedInput | `Engine/Plugins/EnhancedInput/Source/EnhancedInput/` | `Input/` |
| InputCore | `Engine/Source/Runtime/InputCore/` | `Input/` |

## Network

| モジュール | ソースパス | 解析フォルダ |
|-----------|----------|------------|
| Net/Core | `Engine/Source/Runtime/Net/Core/` | `Network/` |
| Net/Common | `Engine/Source/Runtime/Net/Common/` | `Network/` |
| Engine (Replication) | `Engine/Source/Runtime/Engine/Private/Net/` | `Network/` |
| NetworkReplayStreaming | `Engine/Source/Runtime/NetworkReplayStreaming/` | `Network/` |
| Net/Iris | `Engine/Source/Runtime/Net/Iris/` | `Network/`（Iris レプリケーション） |

## Audio

| モジュール | ソースパス | 解析フォルダ |
|-----------|----------|------------|
| AudioMixer | `Engine/Source/Runtime/AudioMixer/` | `Audio/` |
| Engine/Sound | `Engine/Source/Runtime/Engine/Classes/Sound/` | `Audio/` |
| AudioModulation | `Engine/Plugins/Runtime/AudioModulation/` | `Audio/` |
| AudioSynesthesia | `Engine/Plugins/Runtime/AudioSynesthesia/` | `Audio/` |

## GameFramework

| モジュール | ソースパス | 解析フォルダ |
|-----------|----------|------------|
| Engine/GameFramework | `Engine/Source/Runtime/Engine/Classes/GameFramework/` | `GameFramework/` |
| Engine (Private) | `Engine/Source/Runtime/Engine/Private/` | `GameFramework/` |

## Core

| モジュール | ソースパス | 解析フォルダ |
|-----------|----------|------------|
| Core | `Engine/Source/Runtime/Core/` | `Core/` |
| CoreUObject | `Engine/Source/Runtime/CoreUObject/` | `Core/` |

## Niagara

| モジュール | ソースパス | 解析フォルダ |
|-----------|----------|------------|
| Niagara | `Engine/Plugins/FX/Niagara/Source/Niagara/` | `Niagara/` |
| NiagaraCore | `Engine/Plugins/FX/Niagara/Source/NiagaraCore/` | `Niagara/` |
| NiagaraShader | `Engine/Plugins/FX/Niagara/Source/NiagaraShader/` | `Niagara/` |
| NiagaraVertexFactories | `Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/` | `Niagara/` |

## Physics

| モジュール | ソースパス | 解析フォルダ |
|-----------|----------|------------|
| PhysicsCore | `Engine/Source/Runtime/PhysicsCore/` | `Physics/` |
| Chaos | `Engine/Source/Runtime/Experimental/Chaos/` | `Physics/` |
| Engine/PhysicsEngine | `Engine/Source/Runtime/Engine/Classes/PhysicsEngine/` | `Physics/` |

## WorldBuilding

| モジュール | ソースパス | 解析フォルダ |
|-----------|----------|------------|
| WorldPartition | `Engine/Source/Runtime/Engine/Public/WorldPartition/` | `WorldBuilding/` |
| LevelStreaming | `Engine/Source/Runtime/Engine/Private/` | `WorldBuilding/` |
| PCG | `Engine/Plugins/PCG/` | `WorldBuilding/` |

---

## 使い方

1. ユーザーが「GAS の AbilitySystemComponent を調べて」と言ったら、このファイルで GAS のソースパスを確認
2. `D:\UnrealEngine\{ソースパス}` を Glob/Grep で探索
3. 対象システムの `_source_map.md` があればそちらも参照（より詳細なファイル→クラス対応表）
