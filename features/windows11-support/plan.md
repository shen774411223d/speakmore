# Implementation And Validation Plan

## Target

Spec: features/windows11-support/spec.md
Spec Review: features/windows11-support/review.md
Owner: TBD
Last Updated: 2026-10-16
Format Version: 2

Status: Ready For Development

## Context Reviewed

- instructions: AGENTS.md、CLAUDE.md；依赖局部安装、取证后下结论、测试先红后绿、独立审查、只在当前 feature 分支工作，不推送或合并。
- spec: [spec.md](spec.md)，Ready For Development；平台、行为和分期以该文档为准。
- review: [review.md](review.md)，Ready For Development，0 项 Blocking；独立审查与源码复核均完成。
- code/tests/docs: package.json、bin/speakmore.mjs、src/app/main.mjs、helper.mjs、delivery.mjs、setup-state.mjs、keys/、src/control.mjs、src/storage/、src/cli/、src/renderer/bar/、onboarding/、shared/、src/preload/、scripts/build/、scripts/release/、scripts/dev/e2e.mjs、对应 test/、docs/development.md 和 environment-todo.md。
- source baseline: `01bb67ebebf842463e35cb1cf47595aa0353acd0`。本计划仅为规划，全部切片 Pending，尚无 Windows 产品运行证据。

## Summary

- Goal: 先交付一次真实 Windows 听写的可见效果，再完成共享功能、安装版 CLI 和本地安装回退验证。
- Non-Goals: 继承规格的范围排除和 R11 分期；不进行发布、购买证书、移动现有仓库或安装系统工具。
- Primary Risk: 剪贴板事务与输入钩子会影响正在使用的前台应用；必须让失败停在可恢复状态，并在 Windows 本端证明行为，不能依赖假助手 E2E 代替。

## Open Questions

| Question | Owner | Blocks Development | Target Date |
|---|---|---|---|
| None identified | — | No | — |

环境未就绪或测试暂缺并非需求空白：在对应切片的依赖/验收处报告阻塞，不能宣称完成，也不阻塞先写可运行的薄切片。

## Implementation Approach

- 保留现有 Electron renderer、provider、refine、音频、历史和计费；为平台差异建立少量有实际职责的边界，不复制一套业务代码。
- `src/app/platform/` 拥有 Electron 的权限/能力检查、权限入口、窗口参数和系统坐标转换；`src/storage/paths.mjs` 拥有用户数据与运行目录；`src/cli/native-helper.mjs` 统一选择和构建原生助手；CLI 平台进程操作留在 `src/cli/`，不让 renderer 调 Win32。
- `native/windows/` 使用 C++17、Win32、MSVC/Windows SDK。将消息循环/键鼠钩子、剪贴板事务、账户/进程/目录检查分开；用 SDK 的 C++/WinRT `Windows.Data.Json` 解析 JSON，使用系统库而非手写 JSON 解析器或额外引入 FFI。IO 线程向消息循环排队，钩子回调不做剪贴板、磁盘或模型工作。
- 沿用现有助手 JSON 行协议，给 paste 结果增加明确分类和剪贴板状态，至少区分已尝试注入、结果留在剪贴板、保留原剪贴板需从历史取回。旧 macOS 结果通过显式适配保持原行为；Windows 超时/崩溃/未知结果默认禁止上层改剪贴板。`createDelivery` 只负责先落盘与选提示，Windows 剪贴板操作由原生事务独占，移除 Windows 路径中无条件 writeClipboard 的兜底。
- 助手过滤自身注入标记，保留实际按键抬起、自动重复与其他修饰键的区分；模拟 Ctrl+V 前核对前台目标、完整性级别及物理修饰键状态，在现有请求期限内等待安全窗口或降级，不强行释放用户仍按住的键。左右 Ctrl 和 AltGr 分别覆盖测试。
- 剪贴板事务先枚举并完整快照支持格式，再写临时 Unicode 文本，注入并在目标有读取机会后恢复。根据序列号、owner 与本次事务标记共同判断是否仍为本次内容；在同一次 OpenClipboard 排他访问中复核与修改，处理序列号变化/回绕和竞争，不只靠计时猜测。图片按其句柄类型复制，文件列表只处理实体路径，多格式文本保持 HTML/RTF 与纯文本共存；不能完整保存的对象走规格降级。给临时听写设置 Windows 官方剪贴板历史/云同步排除格式，不能承诺第三方管理器一定遵守。
- Windows 控制通道继续使用 node:net，但端点按规范化数据根目录区分。用 node:crypto 生成用户级随机控制密钥，由原生目录检查设置并验证数据/run 目录及密钥文件的用户访问权限；启动失败时不开放控制端点。Windows 连接增加双方带随机 nonce 的 HMAC 身份证明，认证前不发送控制命令或 key；密钥不在协议中明文传送、不写日志。认证错误有界限时并关闭连接。macOS 保持已有 socket 协议与权限逻辑。此扩展对应 R9 允许的等效认证，避免把可猜管道名或默认 DACL 当成隔离。
- 安装版通过独立 C++ console 启动器 `speakmore.exe` 启动同包 `SpeakMore.exe` 的 Electron Node 模式，传入实际 CLI 脚本、stdin/stdout/stderr 和退出码；不依赖外部 node.exe。GUI 再启动时清掉 ELECTRON_RUN_AS_NODE，开发版和安装版使用不同且明确的参数列表。核对打包时的 RunAsNode fuse，并以安装产物验证，不假定 .cmd shim 自带运行时。
- Windows 打包采用新增局部开发依赖 `electron-builder@26.15.3` 与 NSIS，仅用于 Windows；macOS 继续使用现有 `@electron/packager@20.3.0` 和签名链。保留 Electron 44.7.0 与其他业务依赖。先复用 stageApp 的运行依赖和许可证清理能力，去掉对 Unix find/npm 可执行方式的假设；Windows 包包含 CLI、预编译助手、console 启动器与图标，先维持 asar:false 以匹配当前路径。
- builder 采用独立配置、仅 x64、本地输出、禁止 publish；明确按用户安装与标准权限运行，不提供安装到系统目录的隐含提权路径。添加项目局部 NSIS include，扩展数据清理选择与安装备份/失败恢复；不能假定 deleteAppDataOnUninstall 能管理规格自定义数据目录。版本间数据库不变，安全检查失败时拒绝本次操作，保留旧包和数据。
- 新依赖依据：[npm 版本元数据](https://registry.npmjs.org/electron-builder/26.15.3)已核对版本、MIT 许可证和 Node 要求；维护与使用范围依据 [官方仓库](https://github.com/electron-userland/electron-builder)和已检索的 NSIS/contents 官方文档。实施 S5 时生成 package-lock，核对间接依赖许可、漏洞与下载工具校验；本次未安装或完成安全审计，审计阻塞不得绕过。NSIS/下载器的实际版本与摘要进入构建证据，不全局安装工具、不引入另一个更新服务。

Architecture Check:
- Boundary: renderer 只消费平台中性的状态与能力；native 独占 Win32 剪贴板/输入；CLI 独占终端和进程启动；Windows 发行入口不调用 macOS 发布脚本。
- SSOT: 规则见 spec；设置默认值/可选键由 settings 与平台能力共同给出；路径由 paths；paste 结果分类由 helper 契约；人类文案由 src/i18n/locales/；运行与发行版本由 package.json/锁文件。
- Existing Mechanism: 共用 transition、startHelper、createDelivery、safeStorage、控制命令 handlers、DatabaseSync、主题 token 和三类 renderer。
- Interface: 只扩展助手结果、能力与 Windows 控制认证；录音和模型协议、数据库 schema 保持不变。安装与开发入口显式给出平台/架构，不靠文件名猜运行环境。
- Docs: 更新两份 README、docs/development.md、environment-todo.md、助手协议说明与 CLI 帮助；Windows 当前 update 状态不能沿用 macOS 下载说明。
- Locality: 平台代码集中在 app/platform、cli 的原生/启动适配、storage paths 和 native/windows；不把 process.platform 判断撒进 provider 或每个 renderer。
- Design Checks: 避免只加 .exe 而保留不安全的 delivery 兜底、把权限布尔值伪造成 Windows 授权、把安装版 CLI 变成空壳，以及复制整套 UI 的改动扩散。
- Test Target: 状态迁移、助手结果与降级、数据访问隔离、端点认证、真实前台投递、安装后命令行与数据保留。
- Refactor Needed First: 先给路径、能力、原生执行路径与进程启动增加可注入平台依赖；保持 macOS 默认实现与原有测试输入兼容。
- Separate long-term architecture decision doc needed: No，当前决策与后果在本计划足够表达；不新增重复规格。

TDD Planning:
- 可观察的协议、状态、安全分支、路径、同步和 CLI 行为先写失败测试；以最小实现通过后再整理，保留红/绿输出。
- C++ 键盘判定、格式分类和事务状态以独立测试驱动；另用专用 Windows 测试窗口验证真实钩子/剪贴板，不把纯函数通过等同系统 API 已验。
- 字体观感、真实麦克风、外部模型和第三方应用最后用明确设备/版本实测；文档与生成图标不强造镜像实现的单测。

## Key Decisions

| Decision | Rationale | Rejected Alternative |
|---|---|---|
| WSL 唯一源码 + Windows 专用工作副本 | 符合 R12，隔离依赖并让 Windows 行为在本端执行；同步包含内容清单/摘要和路径检查 | 在同一 node_modules 切换平台，或将 Windows npm 默认跑在 UNC 目录，会混淆二进制与构建归属 |
| Windows 先闭合一次真实听写，再补全功能矩阵 | S2 即能看到录音、模型、历史和投递，后续切片收敛兼容性 | 先完成全部打包/CLI 才看见 UI，反馈过晚 |
| C++ 系统助手 + 显式 paste 结果 | 保留已有进程隔离和协议，安全决策由有剪贴板现场信息的一端负责 | JS 异常时无条件复制，或 helper 只返回一个 ok，无法实现 R5 |
| Windows 控制端点双向认证 + 用户目录访问控制 | 既保护命令授权，也避免假服务端套取 key；使用已有 node:crypto，不加第三方密码库 | 可猜管道名、默认权限或单向发送明文控制 token 不能证明接收者身份 |
| console 启动器复用 Electron 的 Node 模式 | CLI 在没有全局 Node 的机器上保持管道、错误码和再启动能力 | 只分发 mjs/npm shim，或让 GUI 子进程吞掉终端输出 |
| Windows builder + NSIS，macOS 工具链保留 | 安装与卸载、用户级集成有成熟基础；固定新工具版本并局部安装 | 为了 Windows 改写现有 macOS 签名发布链，或第一阶段同时接 MSIX/Store |

## Affected Surface

以下新路径为计划新增，不表示文件已存在。

| Area | Files / Modules | Expected Change | Risk |
|---|---|---|---|
| 平台与引导 | src/app/main.mjs、src/app/platform/（新增）、setup-state.mjs、src/preload/preload.cjs、renderer/shared/setup-text.mjs、onboarding/ | 能力状态、Windows 权限入口/窗口控制、平台数据传给页面 | High |
| 原生与构建 | native/windows/（新增）、src/cli/native-helper.mjs、scripts/build/build-native.mjs | C++ 助手、console 启动器、MSVC x64 构建与预编译路径 | High |
| 投递 | src/app/helper.mjs、delivery.mjs、renderer/bar/、src/i18n/locales/ | 安全结果分类、事务/超时降级与 Windows 文案 | High |
| 存储与控制 | src/storage/paths.mjs、legacy.mjs、src/control.mjs、src/app/keys/key-store.mjs | LOCALAPPDATA、ACL 校验、Windows 认证；macOS socket/旧路径保留 | High |
| CLI | bin/speakmore.mjs、src/cli/main.mjs、daemon.mjs、commands/、updater.mjs、storage/settings.mjs | 平台守卫、进程身份、日志跟随、语言、配置/热键、禁用远端更新的真实状态 | High |
| WSL 桥接 | scripts/dev/windows-sync.mjs、windows.ps1（新增）、.machine.env.example、.gitignore | 目录/内容清单、保密配置隔离、依赖安装/构建/启动入口 | Medium |
| 本地安装 | scripts/build/windows-app.mjs、windows-builder.json、windows-installer.nsh（新增）、stage-app.mjs、package.json/lock、assets/ | 独立 Windows 安装包、快捷方式、CLI、启动项、卸载/恢复 | High |
| 验证与文档 | test/、native/windows/tests/、scripts/dev/e2e.mjs、src/app/e2e.mjs、docs/development.md、README 两份、features/windows11-support/ | 测试窗口、UI 自动化、应用矩阵与本地证据说明 | Medium |

## Execution Plan

1. S1：从 WSL 启动可定位修订的 Windows 主窗口
   - Outcome: 一次同步后，Windows 本地副本启动实际 Electron 窗口；缺 key/缺助手能力给出真实状态，不闪退、不假报 ready。
   - Human Review: No (agent-only)
   - Status: Pending
   - Touches: package 平台声明、app/platform、paths、control、native/windows 的目录/身份检查模式与构建骨架、windows-sync.mjs、windows.ps1、平台/同步/认证测试。
   - Dependencies: 用户按 environment-todo 准备 WSL 与 Windows Node、MSVC；无可用 Windows 环境时可以完成离线测试，但本切片不能标完成。
   - Validation: 先红后绿验证 win32/darwin/linux 的受支持入口、SPEAKMORE_HOME、空格/中文路径、同步白名单、同内容重复导出、拒绝源目录/用户根目录/链接逃逸。启用 Windows 控制前必须通过 HMAC 负例、目录权限和伪服务端测试；记录两端修订摘要、平台/版本、Windows 窗口截图和退出码。
   - Recovery: 不移动现有仓库、不更改真实数据目录；同步仅操作带本项目标识的专用输出目录，删除先 dry-run。失败保留旧工作副本，撤回本切片改动即可返回原 macOS 路径。

2. S2：Right Ctrl 完成一次真实听写并贴入普通输入框
   - Outcome: 在 Windows Notepad 中实际单击开始、松手录音、再次按下结束，模型返回结果落入输入框，录音/历史/记账可查。先完成一个真实可见闭环，不等待完整安装器。
   - Human Review: Yes (HITL)
   - Status: Pending
   - Touches: native/windows 的键盘监听和最小安全纯文本事务、helper/delivery 结果契约、main.mjs、录音引导、Windows 热键/i18n、machine/helper/delivery 测试。
   - Dependencies: S1；本人授权 key、可用麦克风和有人操作的 Windows 桌面。HITL 用于本人录音与前台输入确认，不要求逐文件审批。
   - Validation: 先写 no-overwrite/助手崩溃/超时的红例，再实现。C++ 测试窗口验证一次按下/抬起、重复、其他键、600ms 边界和自注入过滤；最小版只接受空或可完整保存的纯文本，其余格式先安全走历史。保留既有状态机测试，在实际 Notepad 跑一次完整链路并核对历史/录音/费用；阶段结果注明普通格式能力尚未齐全，不能宣称 A3 已完成。
   - Recovery: 关闭钩子与临时麦克风，异常后释放捕获；结果从测试数据目录历史取回，旧版本/原用户目录不动。若纯文本事务不安全，回退为明确的历史降级，不绕过 R5 追求演示成功。

3. S3：正常录音时完整处理按钮、Esc 与常用剪贴板
   - Outcome: 悬浮条操作不抢焦点；普通多格式剪贴板按规格恢复，竞争和限制目标安全降级。
   - Human Review: No (agent-only)
   - Status: Pending
   - Touches: native/windows 的鼠标/捕获、clipboard、测试窗口与故障注入；bar capture 坐标、helper/delivery、对应 i18n 和测试。
   - Dependencies: S2；Windows 测试桌面。第三方应用的最后实测在 S6 收口。
   - Validation: unit 覆盖每个 R5 状态分支，native integration 覆盖实体文件列表、DIB/位图、HTML/RTF、多格式与无法快照对象；占用、恢复失败、期间新复制、并发 paste、超时晚回包、父/子进程退出。验证 injected Ctrl 事件不会变成热键。100%/150%/200% 与混合 DPI、任务栏和主屏切换下核对按钮命中与焦点；真实桌面/第三方行为需人工时记录原因。
   - Recovery: 格式级能力检查不通过即保持原剪贴板与历史恢复，不退成不安全自动复制；测试只使用专用窗口和已知样本，不读取或覆盖用户真实剪贴板作为测试数据。

4. S4：补齐 Windows 设置、窗口和源码 CLI 的日常管理
   - Outcome: 在 Windows 本端可管理 key/模型、主题/语言、历史/用量、日志、状态与后台进程，窗口关闭后托盘可恢复，远端更新状态真实。
   - Human Review: No (agent-only)
   - Status: Pending
   - Touches: cli 平台适配、daemon/main/commands/updater、settings、setup-state、onboarding/shared、窗口和托盘配置、safeStorage/paths 测试、UI 测试入口。
   - Dependencies: S2；可与 S3 的独立测试工作交错，使用相同协议 SSOT。
   - Validation: CLI 原有命令/选项矩阵及 JSON 输出，新增 Windows 语言回退和无 tail/ps/security/Bash 路径；GUI/CLI 共用数据根和实例，PID 复用/非本产品进程拒绝结束，其他账户认证失败。Windows DPAPI 保存/重启读取与不可解密负例。UI 自动化覆盖引导、两种语言、三套主题/深浅色、关闭恢复与本地化错误，provider 原始报错不得透出。
   - Recovery: 配置格式/数据库不变，macOS 原默认实现与 key 命名保留；Windows 远端 update 不调用下载/停止进程，只说明阶段能力。无法确认身份时不强杀。

5. S5：在无 Node 的 Windows 上安装并使用 GUI 与 CLI
   - Outcome: 本地未签名 .exe 可安装，开始菜单/托盘/开机启动有效，speakmore 命令独立可用；覆盖安装和回退保留数据。
   - Human Review: Yes (HITL)
   - Status: Pending
   - Touches: electron-builder 精确版本与锁文件、独立 Windows 配置、stage-app、console 启动器、NSIS include、图标、Windows 安装集成测试与说明。
   - Dependencies: S3/S4；先通过新依赖安全/许可证与源码路径审计，再构建；用户准备可重置的干净 Windows 测试环境，不能用真实用户数据测试卸载。
   - Validation: 锁定 builder26.15.3，以本地 `--win nsis --x64 --publish never` 构建并记录下载工具/产物摘要；核对包含项与排除项。console 启动器先用带空格/中文参数、标准输入、管道、大量输出、错误码、Ctrl+C、ELECTRON_RUN_AS_NODE 环境泄漏测试，再验证安装产物。无 Node/SDK/WSL 环境实际执行 CLI start/status/stop/restart；两个本地测试版本覆盖升级/回退，测试运行文件占用、安装中断、数据保留/清理、PATH/快捷方式/启动项残留；拒绝清理符号链接或重解析点外的目录。未签名警告留证，不关闭系统安全功能。
   - Recovery: 安装前核实并停止本产品，保留旧程序备份与本地安装包；新文件在受控临时位置准备，失败恢复旧程序，数据目录独立且不做 schema 迁移。安装自定义逻辑必须通过故障测试，不能依赖默认 NSIS 行为假定事务成立。

6. S6：跨应用与 macOS 回归收口，交付本地可复现证据
   - Outcome: 本轮规格 A1–A8 有明确通过/失败/未触发状态，README/开发说明可由新环境复现，既有 macOS 仍可用。
   - Human Review: Yes (HITL)
   - Status: Pending
   - Touches: 功能/应用/环境矩阵、UI 截图入口、E2E fixture、测试与文档缺口；仅修复本迁移造成的问题。
   - Dependencies: S1–S5；真实 Windows 应用、本人模型 key；macOS 回归必须有 macOS 环境，没有则明确阻塞最终验收，不用 Linux 测试代替。
   - Validation: 按 Coverage Plan 执行共享自动化、Windows 原生/安装/UI 测试及第三方应用矩阵；两家 provider 各实际听写，录音/计费对账。按 AGENTS 独立审查后逐条验证发现，再运行相关回归。完成 vibe-test 证据；只有用户要求提交时再走 vibe-commit，不自动 push、合并或公开发布。
   - Recovery: 保留仍可运行的上一切片和原 macOS 构建入口。未通过项明确列入交付限制与剩余工作，不删除样本证据或声称全面兼容；不得为过门禁关闭失败测试。

各切片按一个逻辑单元准备提交边界，实际提交仍以用户授权为准；前置文件可早于所属切片创建，但切片完成按可观察结果判断。

## Validation Plan

本节命令均为实施后待执行的验证，不表示本次已跑；新增命令由切片提供并先验证帮助/错误退出，再更新文档。

| Acceptance / Risk | Validation Method | Expected Evidence |
|---|---|---|
| A1/R1 平台回归 | 复用 npm run lint、npm run test:unit；macOS 运行既有 test:install。将依赖 Darwin/Bash 的测试归属明确，Windows 有等价项而非静默跳过 | 各平台原始输出、计数、失败/未触发列表，features/windows11-support/evidence/test.md 的脱敏摘要 |
| A2 输入状态/焦点 | node:test 状态机/协议 + Windows 原生 unit/CTest + 专用窗口 integration，S2 起持续执行 | 红/绿日志、按键事件计数、捕获释放、前台窗口身份与 DPI 命中记录 |
| A3 剪贴板与权限 | native API 真集成、事务故障注入、应用矩阵 manual；检查只在安全分支改剪贴板 | 文本/格式摘要前后对比、状态/事件顺序、失败降级录屏；不保存真实剪贴板内容 |
| A4 模型/数据/key | 复用 provider/refine/storage/key tests；Windows safeStorage 实测；隔离 home 下两家真实模型 E2E | 每家输入/输出/计费对应、重启读取、离线/拒绝权限/断设备结果；私有录音与 key 不入库 |
| A5 控制与 CLI | node:test + Windows console/pipe integration；失败身份认证、伪服务端和 PID 复用 fixture | 命令矩阵、stdout/stderr/退出码、跨用户拒绝和无意外副作用日志 |
| A6 安装与回退 | Windows build + 干净环境 installation integration；自定义 NSIS 失败恢复与清理测试 | 产物/依赖摘要、两版本数据对照、无开发工具运行、安装中断与卸载残留清单 |
| A7 WSL 调试链 | windows-sync 单元与跨系统启动 integration/E2E，改源码并验证 Windows 页面中的修订标记 | 源清单/内容摘要、两端平台和运行路径、中文/空格/失败退出码证据，不含本机敏感值 |
| A8 UI/文案 | 复用 renderer 纯函数测试；S4 新增 Electron UI 自动化与 screenshot 入口；对照主题/语言/DPI | 页面状态矩阵、无原始 API 错误、标题栏/拖动/缩放截图和人工视觉复核 |

原始本机日志/录音/截图先留在 gitignored `build/windows-validation/` 或本机配置指定目录；仅经脱敏的计数、命令、版本、摘要与结果汇总进入 feature evidence。S1/S4/S5 提供新增测试入口，不能在入口不存在时伪造执行记录。

## Coverage Plan

- Target Platforms: 直接引用 spec Goal 的 Target Platforms，不另行扩大或缩减。
- Automation Gap: 现有 E2E 用 fakeHelper 且依赖真实模型，没有真实 Win32 输入/安装/UI 自动化。S2 添加 native 测试窗口和确定性 provider fixture；S4 添加局部 UI 驱动；S5 添加安装/console harness。UI 驱动依赖在 S4 选用与现有 Electron/Chromium 验证兼容的精确版本并锁文件，未安装兼容前不把 UI 路径记为覆盖。
- Manual Exceptions: 真实麦克风、两家外部模型、第三方应用与系统桌面/DPI 操作保留人工证据；普通 UI 导航、文案和错误恢复不得因没有现成工具而全部改为手测。关键路径仍需有自动化替身和确定性断言。

| Acceptance / Risk | Layer | Required Coverage | Platform / Device | Existing / New Evidence |
|---|---|---|---|---|
| A1 平台契约 | Unit / Integration | 每个变更公共契约的成功、拒绝、错误分支均有断言；保留原平台测试结果 | 规格平台；WSL 只承接共享纯逻辑 | 原 test/ + 平台分派测试 |
| A2 状态/捕获 | Unit / Integration | KEY_DOWN/UP、重复、组合、长按阈值前/等于/后、丢抬起、崩溃/退出全覆盖；注入过滤与捕获释放 100% 决策覆盖 | Windows 测试桌面 + macOS 既有回归 | machine/helper tests、native/windows/tests |
| A3 剪贴板 | Unit / Integration / Manual | R5 支持/拒绝/竞争/失败结果逐项覆盖；Notepad/Edge/VS Code/Word/微信/Terminal 每项均有结果，无未说明空格 | Windows x64，标准/受限目标 | 原生事务故障测试 + manual 应用矩阵 |
| A4 模型与数据 | Unit / Integration / UI/E2E / Manual | 确定性成功/空音频/模型错误/保存错误自动化；每家真实模型至少 1 次完整听写，未运行明确标记 | Windows 实机 + 共享测试 | 原 server/storage/key tests + 新 Windows E2E |
| A5 认证/进程/CLI | Unit / Integration | 无认证/错密钥/重放/伪服务端/权限拒绝、PID 复用与每个 CLI 命令的关键成功失败路径全覆盖 | Windows；macOS socket 兼容 | control/daemon/CLI tests + launcher harness |
| A6 安装/卸载 | Integration | 全新安装、覆盖升级、退旧版、运行占用、中断、保留/清理数据、路径拒绝各 1 组以上；无 Node 条件必须真实成立 | 干净 Windows x64 | NSIS/安装版本对照及残留断言 |
| A7 开发桥接 | Unit / Integration / UI/E2E | 两端依赖归属、内容修订、无敏感文件、中文/空格、同步/启动失败均自动断言 | WSL2 + Windows 本机 | 同步测试 + Windows 页面修订核对 |
| A8 UI | UI/E2E / Visual | 引导→听写→历史→设置核心路径自动化；两语言×三主题×深浅色的关键页面矩阵，DPI100/150/200 与混合 DPI 有截图/操作证据 | 规格平台 | 新 UI harness + renderer tests + screenshot/manual |

## Coverage Expectations

- Existing coverage command: None identified；现有 node:test 是用例测试，没有全库百分比门槛，不能把 README 的历史通过数量当本轮结果。
- Expected new or updated tests: test/app/helper/delivery/setup-state、test/control、test/storage/paths/settings/key、test/cli/bin/daemon/main/updater、test/scripts/build、test/scripts/dev/windows-sync.test.mjs（新增）、test/renderer 的平台能力与 UI harness，以及 native/windows/tests。不复制测试实现，优先断言用户结果和副作用。
- Coverage threshold or target: 默认变更业务模块 line≥80%、branch≥70%；权限/认证、状态机与剪贴板事务目标 line≥90%、branch≥85%，所有安全决策 100% 用例覆盖。Node 使用该锁定运行时支持的 coverage；C++ 分支以可独立测试状态机/事务分支矩阵记录，若没有可信的原生覆盖率工具则不报百分比已达标，保留工具缺口与逐分支证据。UI 按本节场景矩阵计数，不用行覆盖率替代。

## Rollback / Recovery

- 数据库 schema 不变，macOS 默认路径/签名/发布脚本保留；按切片隔离回退，不通过删除用户目录实现重置。
- Windows 同步只在带本项目标识的专用目标内更新清单拥有的文件；真实主源码和用户数据不能成为清理目标。对重解析点、符号链接、目录身份不符一律拒绝并给出原因。
- 剪贴板与捕获由本次事务/会话拥有；晚回包不再改已结束事务，失败不走 JS 无条件覆盖。保留原内容或从历史取回成品，不能以“已发送”证明“已插入”。
- 安装操作先保留旧程序与本地安装包，数据独立；新版本自检失败或安装中断先恢复旧程序，再输出明确失败。卸载保留数据与清理必须分别测试，并仅操作已验证属于本产品的路径/注册项。
- 任何现有 macOS 行为回退、密钥泄漏、误杀、不可恢复的剪贴板覆盖都阻塞切片完成；停下取证，不通过更高权限或跳过测试掩盖。

## Assumptions

- 用户已决定的 x64、Right Ctrl、剪贴板处理和本地效果优先均为规格规则，不在此重新假设。
- Node、MSVC、Windows 桌面、模型 key 与 macOS 回归机属于执行前置条件；尚未检查安装状况，不能把资源缺失改写为测试通过。Windows 权限、DPAPI、其他账户隔离和 UI 兼容均由测试证明，不假定成立。
- 打包/调试入口与上述新增路径是计划产物，不是当前已经可执行的命令。此计划没有安装软件、实现代码、运行产品测试或公开发布。
