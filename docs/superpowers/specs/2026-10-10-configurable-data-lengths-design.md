# 从站「数据长度（COT/CA/IOA 长度）可配置」设计规范

- 日期：2026-10-10
- 状态：设计完成，待实现
- 范围：从站（slave）底层协议核心库（`iec104sim-core`）、Tauri 后端（`iec104sim-app`）与 Vue 前端（`frontend`）。

---

## 1. 背景与业务价值

当前 IEC 60870-5-104 模拟器中，ASDU 的关键地址与原因长度硬编码为：
- 传送原因（COT）：硬编码为 2 字节（1 字节原因 + 1 字节源发地址 `0x00`）；
- ASDU 公共地址（CA）：硬编码为 2 字节（`u16`）；
- 信息体地址（IOA）：硬编码为 3 字节（`[..3]`）。

在实际电力自动化现场（特别是国内规约转换器、老旧 RTU/DTU/FTU，或由 IEC 60870-5-101 规约迁移的设备）中，常存在：
- COT 长度为 1 字节（无源发地址）；
- ASDU 公共地址为 1 字节（最大站号 255）；
- 信息体地址为 2 字节（最大点号 65535）或 1 字节。

如果双方长度不一致，所有后续字节偏移量将全线错位，导致命令无法识别、上送数据乱码。因此，将这三个长度做成**服务器级独立可配置**具有极高的实用价值。

---

## 2. 核心架构与多服务器隔离决策

1. **服务器级作用域（Per-Server Scope）**：
   - 该配置依附于从站服务器实例（`ServerInstance`），属于 `RemoteOperationConfig` 的一部分；
   - 不同的从站服务器（如端口 2404 与 端口 2405）各自拥有独立的长度配置，互不干扰；
   - 支持从两处入口打开修改：
     - 入口 A：顶部菜单「设置」→「修改运行参数」（作用于当前选中的服务器）；
     - 入口 B：左侧连接树右键具体服务器 →「修改运行参数」（精准作用于目标服务器）。
2. **默认值与合法取值范围**：
   - `cot_len`（传送原因长度）：可选 `1` 或 `2` 字节，默认 `2`；
   - `ca_len`（公共地址长度）：可选 `1` 或 `2` 字节，默认 `2`；
   - `ioa_len`（信息体地址长度）：可选 `1`、`2` 或 `3` 字节，默认 `3`。
3. **向后兼容与持久化保证**：
   - 依赖 Rust `#[serde(default)]`，历史旧配置与未声明新字段的 JSON 在反序列化时自动补齐默认值 2/2/3；
   - 完美适配 `save_config`（导出 JSON）、`load_config`（导入 JSON）以及 `persist_workspace`（关机记忆）。

---

## 3. 技术实现落点

### 3.1 协议核心库 (`crates/iec104sim-core`)
1. **配置模型 (`slave.rs`)**：
   - `RemoteOperationConfig` 增加字段：
     ```rust
     pub cot_len: u8,
     pub ca_len: u8,
     pub ioa_len: u8,
     ```
   - 提供 `Default` 实现：`cot_len: 2, ca_len: 2, ioa_len: 3`。
2. **I 帧生成 (`slave.rs`)**：
   - `build_i_frame`：
     - ASDU 头长 = `Type(1) + VSQ(1) + cot_len + ca_len + ioa_len + value_bytes.len()`；
     - 写入 COT：`cot_len == 1` 写 `cause`；`cot_len == 2` 写 `[cause, 0x00]`；
     - 写入 CA：写入 `&ca[..ca_len as usize]`；
     - 写入 IOA：截取 `&ioa[..ioa_len as usize]`；
     - 动态计算 APCI 控制域总长。
   - `encode_point_frame_ex`、`encode_points_discrete`：IOA 截取长度采用 `ops.ioa_len`。
3. **下行命令解析 (`slave.rs`)**：
   - 控制命令（Type 45-51, 58-64）、总召（100）、电度召唤（101）、时钟同步（103）：
     - 动态计算起始偏移：
       `ioa_start = 6 (APCI) + 1 (Type) + 1 (VSQ) + cot_len + ca_len`；
     - 依据 `ioa_len` 读取 IOA（1字节/2字节小端/3字节小端）；
     - 控制命令值（SCO/DCO/设定值）位于 `ioa_start + ioa_len` 之后。
4. **报文解析日志 (`decode.rs`)**：
   - 扩展支持指定长度解码，避免通信日志显示乱码。

### 3.2 前端界面 (`frontend/`)
1. **类型定义 (`frontend/src/types.ts`)**：
   - `RemoteOperationConfig` 补充 `cot_len: number`、`ca_len: number`、`ioa_len: number`；
   - `DEFAULT_REMOTE_OPS` 补充对应默认值。
2. **表单组件 (`frontend/src/components/RemoteParamsForm.vue`)**：
   - 位于最顶部（**「链路参数」板块正上方**）；
   - 新增 `<section class="rp-sec">`，标题为“数据长度”；
   - 提供三个下拉选择框（Select），带有推荐默认提示。
3. **国际化 (`zh-CN.ts` & `en-US.ts`)**：
   - 提供中英双语标签与说明文案。
