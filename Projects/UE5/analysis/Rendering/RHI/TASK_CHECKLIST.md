# RHI ドキュメント作成チェックリスト

## Overview（1本）
- [x] `12_rhi_overview.md` … RHI 全体像・階層構造・RDG との関係

## Details（グループ別概要 5本）

- [x] `a_rhi_abstraction.md`    … RHI 抽象層（FDynamicRHI / FRHICommandList / IRHICommandContext）
- [x] `b_rhi_resources.md`      … RHI リソース（FRHIResource / Texture / Buffer / PSO）
- [x] `c_rhi_d3d12_device.md`   … D3D12 デバイス層（FD3D12DynamicRHI / Adapter / Device / Queue）
- [x] `d_rhi_d3d12_context.md`  … D3D12 コマンド実行（ContextCommon / CommandList / Submission）
- [x] `e_rhi_rdg_bridge.md`     … RDG → RHI ブリッジ（FRDGBuilder からの変換フロー）

## Reference（ファイル別 6本）

### 抽象層（3本）
- [x] `ref_rhi_commandlist.md`   … FRHICommandListBase / FRHICommandList / FRHICommandListImmediate
- [x] `ref_rhi_resources.md`     … FRHIResource / FRHITexture / FRHIBuffer / FRHIUniformBuffer
- [x] `ref_rhi_dynamic_rhi.md`   … FDynamicRHI / IRHICommandContext / グローバル RHI 関数

### D3D12 実装層（3本）
- [x] `ref_rhi_d3d12_device.md`     … FD3D12DynamicRHI / FD3D12Adapter / FD3D12Device / FD3D12Queue
- [x] `ref_rhi_d3d12_context.md`    … FD3D12ContextCommon / FD3D12CommandList / FD3D12CommandAllocator
- [x] `ref_rhi_d3d12_resources.md`  … FD3D12Resource / FD3D12Buffer / FD3D12Texture / Descriptors

---

## Phase 3（コード実行フロー + callout 追加） ← 未着手

### Overview（1本）
- [ ] `12_rhi_overview.md` … `## コード実行フロー` 追加

### Details（5本） — `## コード実行フロー` + `## 関連リファレンス` 追加
- [ ] `a_rhi_abstraction.md`
- [ ] `b_rhi_resources.md`
- [ ] `c_rhi_d3d12_device.md`
- [ ] `d_rhi_d3d12_context.md`
- [ ] `e_rhi_rdg_bridge.md`

### Reference（6本） — `> [!note]-` callout × 3 追加
- [ ] `ref_rhi_commandlist.md`
- [ ] `ref_rhi_resources.md`
- [ ] `ref_rhi_dynamic_rhi.md`
- [ ] `ref_rhi_d3d12_device.md`
- [ ] `ref_rhi_d3d12_context.md`
- [ ] `ref_rhi_d3d12_resources.md`
