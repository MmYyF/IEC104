# IEC 60870-5-104 模拟器开发与项目指南 (GEMINI.md)

本项目为基于 **Rust (Tokio / Tauri 2)** 和 **Vue 3 (TypeScript / Vite)** 构建的跨平台 IEC 60870-5-104 仿真工具集（包含从站仿真器 `IEC104Slave` 与主站客户端 `IEC104Master`）。

---

## 一、项目核心开发协作规则

1. **命令执行权限规范**：
   - **查看与查找类命令**（如 `ls`、`git status`、代码检索、编译状态检查、测试等只读类操作）：AI 助手直接执行，无需向用户反复确认。
   - **代码修改类操作**（修改已有代码、重构、功能新增等）：**必须先向用户说明修改思路和方案，获得用户明确确认后再行动**。
2. **结合 Rust 语法教学**：
   - 开发者目前处于了解 Rust 基本语法的阶段，因此在分析问题、提出修改方案和编写代码时，AI 助手需同步讲解涉及到的 Rust 核心语法、设计模式与异步并发机制，辅助开发者系统性学习 Rust。
3. **本地 Git 提交流程**：
   - 本项目已初始化本地 Git 仓库（分支：`master`）。
   - 每完成一个功能并通过用户测试/确认后，再进行本地 Git 提交（保持颗粒度清晰、提交信息明确）。

---

## 二、开发环境搭建 (Ubuntu x86_64)

### 1. 系统底层依赖库
Ubuntu 22.04 / 24.04 构建 Tauri 2 与 WebKit 桌面窗口所需依赖：
```bash
sudo apt-get update
sudo apt-get install -y build-essential pkg-config perl libssl-dev \
  libwebkit2gtk-4.1-dev libappindicator3-dev librsvg2-dev patchelf
```

### 2. Node.js & npm
- Node.js 版本推荐：`^20.19.0 || >=22.12.0`（当前开发机版本：Node v24.13.0 / npm 11.11.1）
- 安装前端依赖：
  ```bash
  (cd frontend && npm ci)
  (cd master-frontend && npm ci)
  ```

### 3. Rust 与 Tauri 工具链
- Rust 工具链：当前开发机为 `rustc 1.96.1`。
- 安装 Tauri 2 原生 CLI：
  ```bash
  cargo install tauri-cli --version '^2' --locked
  ```
  安装完成后，可使用 `cargo tauri --version` 验证。

### 4. Linux 跨用户图形显示 (X11 / Display)
如果终端开发用户为 `rustdev`，而当前物理屏幕/桌面登录的是普通用户（如 `mayfei`）：
1. 在当前桌面用户的终端中开放显示授权：
   ```bash
   xhost +SI:localuser:rustdev
   # 或允许本机所有本地用户投屏：
   xhost +local:
   ```
2. 在 `rustdev` 的终端环境中配置环境变量：
   ```bash
   export DISPLAY=:0
   ```
   *(或者在桌面环境下直接切换登录到 `rustdev` 用户桌面)*

---

## 三、编译与运行方法

### 1. 开发调试模式（支持前端 Vue 热重载与 Rust 增量重编）
在桌面有图形界面的环境中打开两个终端：

- **从站 (IEC104Slave)**：
  ```bash
  cd crates/iec104sim-app
  cargo tauri dev
  ```
  *(自动在后台拉起 Vite 监听 5176 端口，同时启动桌面 GUI)*

- **主站 (IEC104Master)**：
  ```bash
  cd crates/iec104master-app
  cargo tauri dev
  ```
  *(自动在后台拉起 Vite 监听 5177 端口，同时启动桌面 GUI)*

### 2. 独立发布版编译（Release 二进制）
Tauri 的 Release 模式会将前端编译生成的静态网页（`dist/`）直接内嵌到单个二进制文件中，运行时不依赖 Node.js 或任何本地端口：

```bash
# 1. 编译前端静态网页
(cd frontend && npm run build)
(cd master-frontend && npm run build)

# 2. 编译 Rust Release 二进制（体积约 27-29MB）
cargo build --release

# 3. 直接运行内嵌网页的独立二进制
./target/release/iec104sim-app       # 运行从站
./target/release/iec104master-app    # 运行主站
```

### 3. 本地打包为安装包（.deb / .AppImage）
```bash
(cd crates/iec104sim-app && cargo tauri build)
(cd crates/iec104master-app && cargo tauri build)
```
产物输出路径位于：
- `target/release/bundle/deb/`
- `target/release/bundle/appimage/`

### 4. 后续发布 Windows 版本策略
- **首选（CI 自动编译）**：利用现有的 GitHub Actions 工作流（[`.github/workflows/release.yml`](.github/workflows/release.yml)），在仓库打 Tag 时，GitHub 的 `windows-latest` 机器会自动构建出 Windows `.exe` 安装包、`.msi` 和便携版 `.portable.exe`。
- **本地编译**：在 Windows 设备安装 Visual Studio C++ 构建工具、Node.js、Rust 与 WebView2，执行相同的 `cargo tauri build` 即可。

---

## 四、项目工程结构导航

- `crates/iec104sim-core/`：IEC 104 协议底层库（报文编解码、APDU、COT、ASDU、从站/主站网络状态机、TLS 支持）。
- `crates/iec104sim-app/`：从站 Tauri 后端（60+ Tauri Commands，点表管理、变位模拟、CSV/JSON 回放）。
- `crates/iec104master-app/`：主站 Tauri 后端（多 CA 支持、SOCKS5 代理、总召与控制命令下发）。
- `frontend/`：从站 Vue 3 前端界面。
- `master-frontend/`：主站 Vue 3 前端界面。
- `shared-frontend/`：主从站共享前端代码（通用 UI 组件、多语言 i18n、报文解析器、样式变量）。
