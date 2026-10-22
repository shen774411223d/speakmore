# Windows 11 / WSL2 开发准备清单

关联规格：[spec.md](spec.md)。本文件是环境准备和资料取证说明，不是实施计划，也不代表当前代码已经支持 Windows。按规格当前迭代先取得本机运行效果，再验本地安装包；正式签名与远端发布不阻塞本轮。

结论：**在 WSL2 中编辑源码、管理 Git 和运行平台无关检查；在 Windows 本地调试工作副本中安装 Windows Electron，并用 Windows 工具链运行、编译和打包。Electron 作为项目局部依赖安装，不需要全局安装。** 该安排落实规格 R12。

## 1. 工具与文件放在哪里

| 内容 | WSL2 | Windows 11 |
|---|---|---|
| 主源码和 Git | Linux 文件系统中的唯一编辑源，例如 `~/projects/speakmore` | 不在调试副本反向编辑业务代码 |
| 调试工作副本 | 单向输出源码清单 | 本地 NTFS 上的专用目录，由本机配置指定；不使用 WSL UNC 路径作为 npm 工作目录 |
| Node/npm | Linux Node，用于共享逻辑检查 | Windows Node，用于 Windows 安装、启动和打包；不能误用 WSL Node |
| Electron | 平台无关检查不需启动 Linux Electron；可选 WSLg 辅助检查另行准备 | **调试副本自己的 `node_modules/electron`，二进制为 `electron.exe`** |
| 其他依赖 | WSL 的项目局部 `node_modules` | Windows 副本独立的项目局部 `node_modules`，不能复制 WSL 依赖 |
| 原生助手工具链 | 编辑 C++ 和协议，运行可跨平台的检查 | Visual Studio Build Tools 的 C++ 桌面工作负载、MSVC、Windows SDK；需要时使用随其安装的 CMake 工具 |
| 编辑器 | 通过 WSL 扩展运行项目服务 | 可使用 Windows VS Code + WSL 扩展；不强制更换现有编辑器 |
| 构建和调试产物 | 不进入源码提交 | 保留在本机工作副本或专用输出目录，由实施脚本管理 |

Node 24 是符合当前项目 `engines` 与 `node:sqlite` 使用要求的候选主版本；实施时在两端固定经过验证的补丁版本。Node 20 不满足当前项目要求。Node 工具链可以用用户级版本管理器准备，npm 包全部局部安装，不使用 `sudo npm` 或 `npm install -g electron`。

Windows `node_modules/electron` 是开发依赖位置；Electron 下载缓存与最终安装的 SpeakMore 是另外两件事。缓存不作为运行路径，最终用户只安装打包后的应用，不需要手动安装 Electron。

## 2. 你现在要准备的待办

1. [ ] **Windows：确认系统与 WSL2。** 在 PowerShell 执行下面的只读检查，确认系统架构与发行版；如果 WSL 已经可用，不必重装。

   ```powershell
   wsl --status
   wsl --list --verbose
   [System.Runtime.InteropServices.RuntimeInformation]::OSArchitecture
   ```

2. [ ] **WSL：选定主源码位置。** 使用 Linux 文件系统中的项目目录，从 WSL 打开编辑器。现有目录如需迁移，先备份工作区与 Git 未提交内容，再做可逆迁移；本任务没有移动或删除现有项目。

3. [ ] **两端：分别准备兼容的 Node/npm。** 用已有用户级版本管理器选择 Node 24；未装的那一端再安装对应 Windows 或 Linux 工具链。先执行以下检查，不要立即修改全局配置或卸载其他项目使用的版本。

   WSL Bash：

   ```bash
   command -v node
   command -v npm
   node -p 'JSON.stringify({platform:process.platform,arch:process.arch,node:process.version})'
   npm --version
   ```

   Windows PowerShell：

   ```powershell
   Get-Command node, npm.cmd
   node -p 'JSON.stringify({platform:process.platform,arch:process.arch,node:process.version})'
   npm.cmd --version
   ```

   预期两端分别为 `linux` 与 `win32`，不是“两端指向同一个 Node”。具体个人路径只存本机配置，不写进仓库。

4. [ ] **Windows：准备原生构建工具。** 从 [Visual Studio 官方下载页](https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio)安装 Build Tools，选择“Desktop development with C++”与 Windows SDK。通过对应 Developer PowerShell/命令提示符检查 `cl`；普通终端查不到不等于未安装。NSIS 与打包 npm 工具后续按项目版本准备，不必现在全局安装。

5. [ ] **Windows：准备真实输入测试。** 确保麦克风可用、桌面应用可访问麦克风；准备自己的 Gemini/OpenAI key，通过应用输入或本机受保护配置注入。测试应用按规格 A3 准备；暂缺付费 Office 时记录缺口，不把未运行写成通过。

正式发布准备已移出本轮待办；后续进入发布阶段时再落实 Windows 签名身份与远端更新入口，范围以规格 R11 为准。

## 3. 等平台适配完成后再执行

下面是未来操作顺序，不是当前分支已有能力。**现在不建议直接运行 `npm ci`、`npm start` 或套用 Forge 的跨平台安装命令来启动 Windows 版**：当前 `os` 仅声明 `darwin`，部分 npm 版本/安装情形可能拒绝非 macOS 平台；即使安装成功，默认助手构建和权限调用也仍是 macOS 实现。具体表现需当时实测，不使用 `--force` 绕过。

- [ ] 由项目提供 WSL → Windows 的单向源码导出/同步入口。同步锁文件和运行时所需资源；隔离两端依赖与构建目录，排除 `.git`、key、录音、历史、`.env`、`.machine.env` 和其他本机文件。只操作专用工作副本，对移除陈旧文件先给出清单，不能递归清理任意用户目录。
- [ ] Windows 工作副本产生后，在 **Windows PowerShell 的该目录**执行 `npm.cmd ci`；由项目锁定版本准备 Electron 与打包工具，不额外安装最新版 Electron。若二进制采用延迟下载，则按该锁定版本支持的机制下载；不能据当前最新文档猜测 44.7.0 的安装时机。
- [ ] WSL 在主源码目录独立安装开发依赖并运行平台无关检查；是否延迟/跳过不使用的 Linux Electron 下载由实施脚本决定。不把 Windows `electron.exe` 写进 Linux 依赖树来兼顾两端。
- [ ] 由项目提供 Windows helper 构建、Windows Electron 启动、DevTools 调试和 .exe 安装器构建入口。具体命令在实施完成并验证后写回开发文档；本清单不虚构尚不存在的 `npm run dev:win` 等命令。
- [ ] 日常循环为“WSL 编辑 → 单向同步 → Windows 构建/启动 → Windows 前台应用实测”。可从 WSL 调用 `powershell.exe` 发起，但必须设置 Windows 工作目录、校验退出码和使用 Windows Node，不把调用发起地当成执行平台。
- [ ] 不把远程调试端口绑定到公网，不把 key、音频或结果正文写进调试日志。调试数据目录与真实用户数据分离。

依赖安装完成后，可以在 Windows 工作副本中检查局部 Electron：

```powershell
node -p "require('electron/package.json').version"
node -p "require('electron')"
```

应得到锁定版本与该副本中的 Windows `electron.exe` 路径；若二进制尚未下载，按该版本机制完成后再复查。不要用可能临时拉取其他版本的无约束 `npx electron` 作为验证依据。

## 4. 调研证据与适用边界

资料核对日期：2026-10-16。以下为已读取的官方原文；环境布局是结合这些约束为本项目作出的工程选择，并非声称官方规定必须维护两个仓库。

| 来源 | 原文要点 | 对本项目的含义 |
|---|---|---|
| [Electron Installation](https://www.electronjs.org/docs/latest/tutorial/installation)；[读取的官方源文](https://github.com/electron/electron/blob/main/docs/tutorial/installation.md) | 推荐项目局部安装；按平台与架构取得二进制；缓存位置与平台相关。 | 使用仓库锁定版本，两端依赖隔离。最新 main 的安装/下载时机可能不同于 44.7.0，不照搬新版本细节。 |
| [Electron First App](https://github.com/electron/electron/blob/main/docs/tutorial/tutorial-2-first-app.md)（官方文档检索） | 入门教程仍明确提醒 Windows 用户避免用 WSL 直接跟随启动教程。 | 不能把“在 WSL npm start”作为 Windows 功能默认验证路径；这不是禁止用 WSL 编码。 |
| [Electron Forge Developing with WSL](https://github.com/electron/forge/blob/main/docs/guides/developing-with-wsl.md) | WSL 可通过互操作运行 Windows Electron；示例使用平台覆盖安装。还明确警告部分发行工具错误使用 Wine，建议在 WSL 外执行 package/make/publish。 | 证实 Windows 二进制可从 WSL 发起；本项目选择 Windows Node/工作副本来调试与发行。文中 Linux GUI 不支持的描述标注为 2021 年，已不能作为今天的结论。当前项目未使用 Forge，示例不等于本项目命令。 |
| [Microsoft WSL 文件系统与互操作](https://learn.microsoft.com/windows/wsl/filesystems)；[读取的官方源文](https://github.com/MicrosoftDocs/WSL/blob/main/WSL/filesystems.md) | Linux 工具优先使用 Linux 文件系统，Windows 工具优先使用 Windows 文件系统；WSL 可调用 Windows `.exe`。 | 主源码留在 WSL；频繁 npm、原生构建和发行操作在 Windows 本地副本执行。避免跨文件系统的大量依赖 IO。 |
| [Microsoft Node.js on WSL2](https://learn.microsoft.com/windows/dev-environment/javascript/nodejs-on-wsl)；[读取的官方源文](https://github.com/MicrosoftDocs/windows-dev-docs/blob/docs/hub/dev-environment/javascript/nodejs-on-wsl.md) | 建议 Node 与项目文件位于相同操作系统的文件系统；Windows VS Code + WSL 扩展将编辑器 UI 与 Linux 项目服务分开。 | WSL 与 Windows 各有本端工具链；编辑器所在平台不代表应用运行平台。 |
| [Microsoft Linux GUI apps](https://learn.microsoft.com/windows/wsl/tutorials/gui-apps)；[读取的官方源文](https://github.com/MicrosoftDocs/WSL/blob/main/WSL/tutorials/gui-apps.md) | 当前 WSL2 支持 X11/Wayland Linux GUI，并可整合 Windows 桌面与剪贴板。 | WSLg 能显示 Linux GUI，但没有因此把 Linux 应用变成 Win32 应用，不能证明 Windows 钩子、输入权限、DPAPI 或安装更新行为。 |
| [Microsoft Clipboard Formats](https://learn.microsoft.com/windows/win32/dataxchg/clipboard-formats)；[读取的官方源文](https://github.com/MicrosoftDocs/win32/blob/docs/desktop-src/dataxchg/clipboard-formats.md) | 同一次复制可含多种格式，应用可注册 HTML/RTF 等格式；私有格式有不同的资源所有权，系统可转换部分图片格式。 | 不能把原剪贴板仅存成文本，或将任意数据句柄当字节复制；规格 R5 明确普通格式与无法安全快照对象的处理边界，具体实现仍需测试。 |

通用搜索服务本次因审批服务故障未能使用；以上证据来自官方文档检索与公开官方源文读取。未取得独立社区实测，也没有实际安装 Electron、编译 Windows 助手或运行安装器；不将资料调研等同于本机验证。

## 5. 环境就绪检查

- [ ] 两端检查日志明确显示 `linux` / `win32`、架构、Node、锁文件和 Electron 版本；敏感个人路径不进入公开材料。
- [ ] WSL 一次源码修改经同步后，Windows 运行的是同一份修订；同步不会复制依赖、覆盖凭据或误清理其他目录。
- [ ] Windows helper 能触发真实录音与投递，取消后不会卡住 Esc 或鼠标；结果和录音可在历史查到。
- [ ] Windows 干净虚拟机或实体机只安装 .exe 即可启动，无需 WSL、Node、Electron 或 SDK；虚拟机负责安装验证，宿主机再测真实麦克风与前台应用。
- [ ] 把热键、剪贴板与 WSL → Windows 开发链加入实施测试计划，按规格 A2/A3/A7 收集证据；未实测项不写成已通过，也不再列为待用户回答的问题。
