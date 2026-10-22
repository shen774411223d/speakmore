# Windows 11 迁移规格审查

## Blocking Items

- None identified

## Suggested Additions

- None identified

## Conclusion

Target: spec
Conclusion: Ready For Development
Confidence: High

规格的产品范围、平台、交互、隐私、失败降级与本地发行边界足够明确，可以进入实施规划；当前代码与目标之间的差距属于实现任务，没有需要用户再次决策的规格缺陷。

审查对象：[spec.md](spec.md)，审查时状态为 In Review；内容 SHA-256 为 `bf96a33bfb6b3edf66f051ec602f297e6a284529f1e9c245b9bf719244b4f1a8`。状态推进为 Ready For Development 后摘要会变化，以上摘要标识本轮实际审查输入。相关材料：[environment-todo.md](environment-todo.md)。源码基线为 `01bb67ebebf842463e35cb1cf47595aa0353acd0`。

已逐条核实独立审查意见，以下是应进入计划的实现注意事项，不构成待用户决策的问题：

- `src/app/helper.mjs:38` 仅透传 ok/reason，`src/app/delivery.mjs:56` 将所有否定结果映射为 macOS 权限问题，`:62` 的异常分支还会无条件改剪贴板。R2/R5 已授权必要的契约扩展，需将结果分类、Windows 文案和禁止覆盖的降级路径一同实现并测试。
- `src/renderer/bar/machine.mjs:9` 的长按阈值为 600 ms，`:40`/`:48` 的开始与停止均由按下触发，符合 R3。Windows 助手需过滤自身模拟粘贴事件，并验证 Right Ctrl 组合键及 AltGr；这属于 A2/A3 的实现验收。
- `bin/speakmore.mjs:14` 的平台守卫、`src/cli/daemon.mjs` 的 Electron 路径与进程识别、`scripts/build/mac-app.mjs:47` 的 CLI 排除规则都必须适配。R10 要求的是安装后真实可用的 CLI，不能只分发 .mjs 或依赖全局 Node 的 npm shim。

## Effort Estimate

Estimated Effort: Large
Estimate Confidence: Medium
Rationale:
- 涉及原生输入助手、Electron 宿主、共享投递协议、控制通道、CLI、安装回滚及两端测试，跨越多个模块。
- Windows 真实应用兼容性与剪贴板对象生命周期尚未实测，不给出缺乏证据的工时承诺；优先交付薄的真实听写链。

## Validation Notes

- Target context: 已完整读取 spec、环境清单、AGENTS.md/CLAUDE.md、用户 Q1–Q4 答复，检查状态机、助手通信、投递、设置/引导、数据/key、控制通道、CLI、打包和现有测试。另以独立只读审查交叉核实，未直接采信未经源码验证的 finding。
- Structure validation: `python3 <vibe-spec>/scripts/validate_spec.py features/windows11-support/spec.md` 实际输出 `VALID: 0 errors, 0 warning(s)`。
- Contract fit: 复用既有状态机、JSON 行消息、provider、历史及主题；R2 允许为 R5 的安全降级扩展结果，命名管道和 Windows 启动器由 R9/R10 的现有能力缺口支持。
- Acceptance/test coverage: A1–A8 均引用规则并要求可观察证据。原生热键、剪贴板、WSL/Windows 调试和本地安装是待实施/待运行测试，不冒充已验证，也不被误归为未决产品需求。
- Open questions: 当前无阻塞或未解的产品问题；x64、Right Ctrl、剪贴板安全优先、跨环境测试及公开发布后置均已决策。R1/R10 与 R11 的分期引用一致。

## Residual Risk

- Windows 原生钩子、DPI、剪贴板格式、DPAPI/ACL、CLI 启动和安装器未实测；规划通过不等于产品通过。
- 现有 E2E 替换真实助手与投递，只能覆盖部分应用链，不能替代 Windows 原生输入集成测试。
- macOS 回归必须在 macOS 执行；本次未运行产品测试、调用真实模型、构建或安装任何程序。公开签名与远端更新按 R11 后置。
