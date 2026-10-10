# 从站「数据长度（COT/CA/IOA 长度）可配置」实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现 IEC 60870-5-104 从站的传送原因（COT）、ASDU公共地址（CA）与信息体地址（IOA）字节长度的服务器级灵活配置，并在前端「修改运行参数」面板最上方增加「数据长度」配置区。

**Architecture:** 
- 后端：在 `RemoteOperationConfig` 中新增 `cot_len: u8`（1/2）、`ca_len: u8`（1/2）、`ioa_len: u8`（1/2/3），默认分别为 2, 2, 3。动态重构 `slave.rs` 的组包与命令解析。
- 前端：在 `RemoteParamsForm.vue` 最上方（链路参数上方）增加「数据长度」表单模块，支持每台服务器独立配置。

**设计依据：** `docs/superpowers/specs/2026-10-10-configurable-data-lengths-design.md`

---

## 任务拆解清单

### Task 1: 协议核心库扩展配置模型与单元测试
- [ ] **Step 1:** 在 `crates/iec104sim-core/src/slave.rs` 中为 `RemoteOperationConfig` 增加 `cot_len`、`ca_len`、`ioa_len` 字段与默认值实现。
- [ ] **Step 2:** 编写针对配置默认值及 serde 序列化/反序列化兼容性的单元测试。
- [ ] **Step 3:** 运行 `cargo test -p iec104sim-core` 确保模型测试通过。

### Task 2: 改造从站下行命令解析与上行 I 帧编码
- [ ] **Step 1:** 改造 `slave.rs` 中的 `build_i_frame`、`encode_point_frame_ex`、`encode_points_discrete`，动态应用配置中的 `cot_len`、`ca_len` 和 `ioa_len`。
- [ ] **Step 2:** 改造下行命令解析（控制命令 Type 45-51/58-64，总召 Type 100，电度召唤 Type 101，时钟同步 Type 103），将硬编码的偏移量改为动态计算偏移。
- [ ] **Step 3:** 编写自定义 1 字节 COT / 2 字节 IOA 的集成测试（在 `crates/iec104sim-core/tests/` 中），验证从站正确组包与命令响应。
- [ ] **Step 4:** 运行 `cargo test -p iec104sim-core` 验证所有现有测试与新测试通过。

### Task 3: 前端数据模型与国际化支持
- [ ] **Step 1:** 在 `frontend/src/types.ts` 中为 `RemoteOperationConfig` 增加 `cot_len`、`ca_len`、`ioa_len` 字段与 `DEFAULT_REMOTE_OPS` 默认值。
- [ ] **Step 2:** 在 `frontend/src/i18n/locales/zh-CN.ts` 和 `en-US.ts` 中补充「数据长度」相关的中英文文本字典。

### Task 4: 前端界面表单添加「数据长度」板块
- [ ] **Step 1:** 在 `frontend/src/components/RemoteParamsForm.vue` 模板最上方（链路参数上方）增加 `<section class="rp-sec">` 数据长度配置块。
- [ ] **Step 2:** 绑定 `ops.cot_len`、`ops.ca_len`、`ops.ioa_len`，提供 3 个下拉选择框（Select），标注推荐默认项。
- [ ] **Step 3:** 执行 `npm --prefix frontend run build` 确保 TypeScript 类型检查和前端打包完全无错。

### Task 5: 全量验证与本地 Git 提交
- [ ] **Step 1:** 运行 `cargo test --workspace` 确保整套工作区所有测试绿灯。
- [ ] **Step 2:** 验证 JSON 配置文件的保存与加载（`save_config` / `load_config`）。
- [ ] **Step 3:** 呈送测试结果给用户确认，确认后提交至本地 Git。
