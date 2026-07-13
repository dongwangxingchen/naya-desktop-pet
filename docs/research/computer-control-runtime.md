# 电脑控制能力包候选

电脑控制不应由 Godot 主程序临时拼一个自动点击脚本。Godot 只负责角色、权限 UI 和任务状态；真正的终端与桌面执行放在按需下载、独立签名的本地组件中。

## 候选组件

| 项目 | 用途 | 许可证 | 当前判断 |
|---|---|---|---|
| [Open Interpreter](https://github.com/openinterpreter/openinterpreter) | 本地代理、终端、工具编排、模型适配 | Apache-2.0 | 最适合作为执行核心候选 |
| [Cua](https://github.com/trycua/cua) | macOS/Windows/Linux 桌面操作、沙箱与驱动 | MIT | 最适合作为 GUI 控制层候选 |
| [agent-browser](https://github.com/vercel-labs/agent-browser) | 面向代理的浏览器 CLI | Apache-2.0 | 浏览器层优先候选，Rust 便于独立打包 |
| [browser-use](https://github.com/browser-use/browser-use) | 成熟的浏览器代理框架 | MIT | 能力强，但 Python 依赖较重，作为备选 |
| [UI-TARS Desktop](https://github.com/bytedance/UI-TARS-desktop) | 多模态 GUI、浏览器和终端代理 | Apache-2.0 | 体积和模型要求偏重，不做第一版核心 |
| [Agent-S](https://github.com/simular-ai/Agent-S) | 研究型通用电脑代理 | Apache-2.0 | 研究价值高，第一版集成风险偏大 |

## 初步组合

```text
NayaPet / Godot
  │  结构化任务 + 权限决定 + 状态事件
  ▼
本地 IPC（版本化协议）
  ▼
NayaControlRuntime
  ├── Open Interpreter：任务编排与终端
  ├── Cua：桌面 GUI 驱动
  └── agent-browser：浏览器操作
```

这只是技术验证优先级，不代表现在就复制或绑定全部代码。每个项目都必须固定版本、保留许可证，并确认能否被裁剪成签名、公证、可卸载的 macOS 用户级组件。

## 第一轮验证边界

1. 只读查看一个用户选择的文本文件。
2. 只读读取当前进程列表。
3. 打开浏览器并读取用户指定页面标题。
4. 在测试应用中完成一次鼠标点击，但必须经过第二档审批。
5. 关闭电脑控制后，组件进程退出且不保留后台守护程序。

验证期间不允许：

- 默认扫描整个磁盘。
- 让辅助组件长期保存 API Key。
- 绕开桌宠的权限策略直接执行模型输出。
- 自动申请全部 macOS 高危权限。
- 使用 root、LaunchDaemon 或不可卸载的系统修改。

## 生产前门槛

- 下载包具有版本、SHA-256、签名身份和来源清单。
- 固定 Bundle ID、Team ID 与安装位置，避免系统权限反复失效。
- IPC 消息有白名单、尺寸限制、超时和审计编号。
- 第三档完全允许仍保留紧急停止、操作日志和禁止修改自身权限策略的边界。
- 所有上游许可证与修改记录进入第三方清单。
