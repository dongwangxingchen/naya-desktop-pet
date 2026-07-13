# 开发与发布前提条件

## 当前电脑已经具备

实机核对结果：

- macOS `26.5.1`，Apple Silicon `arm64`。
- Godot `4.6.3 stable`。
- 与编辑器匹配的 `4.6.3.stable` 导出模板。
- Git 与 GitHub CLI，仓库已公开。
- Rosetta 已安装，可做 x86_64 兼容性初步验证。

这台电脑足够开始角色技术样机和 Godot 主程序开发。

## 当前缺少

- 完整 Xcode；当前只有 Command Line Tools。
- Apple Developer Program 账号对应的 `Developer ID Application` 签名身份。
- 最终角色的可绑定分层母版。
- Live2D、Spine 或纯 Godot 原生绑定路线的最终决定。

因此现在可以开发和制作本机测试包，但还不能发布普通用户双击即可无警告打开的正式 DMG。

## 建议锁定的基础环境

| 项目 | 建议 |
|---|---|
| 引擎 | Godot 4.6.3 stable，编辑器与导出模板严格同版本 |
| 渲染器 | Compatibility；为透明窗口与低功耗桌面应用优先优化 |
| macOS 最低版本 | macOS 13 Ventura |
| 架构 | Universal 2：`arm64 + x86_64` |
| 发行渠道 | GitHub Releases 的签名、公证 DMG |
| 开发语言 | GDScript 为主；系统桥接使用 Swift/Objective-C 或 GDExtension |
| 本地密钥 | macOS Keychain |
| 开机启动 | `SMAppService`，用户主动开启 |
| 配置与记忆 | Application Support 中的本地数据库；密钥不进数据库 |

完整版本不以 Mac App Store 为首发目标。App Store 沙盒与“按需下载电脑控制组件、执行终端、跨应用操作”冲突。未来可以另做不带电脑控制的 Lite 版。

## 美术前提

正式母版不能只是一张成品图，至少需要以下可编辑图层：

- 后发、脸底、前发。
- 左右眼白、左右瞳孔、上眼皮、下眼皮、眉毛。
- 独立嘴型或口型参数所需图层。
- 头部、颈部、躯干、上臂、前臂、手、腿和靴子。
- 披肩、上衣、裙装、腰带、箭袋、蓝羽饰、头带与珠饰。
- 左右辫子及适合物理摆动的分段。
- 关节和遮挡处被成品图挡住的隐藏区域补画。

所有图层使用同一画布、原点、缩放和命名规范。必须保存 PSD/PSB 或等价分层源文件，以及 Live2D `.cmo3`、Spine `.spine` 或 Godot 绑定场景，不能只保留导出结果。

## macOS 窗口前提

角色窗：

- 小尺寸透明窗口，不做覆盖整个桌面的全屏透明窗。
- Borderless、透明背景、置顶、无焦点。
- 每个大姿势有稳定、简化的鼠标命中轮廓。

管理窗：

- 设置、聊天记录、长期记忆和权限管理使用独立普通窗口。
- 不在角色透明窗内承载复杂下拉菜单或长文本输入。

必须测试：

- 菜单栏与 Dock 的可用区域。
- 屏幕在左、右、上、下排列时出现的负坐标。
- Retina 1×/2×缩放与外接显示器。
- 休眠唤醒、插拔显示器和切换 Space 后的位置与尺寸。
- 角色、气泡、菜单和管理窗的固定层级策略。

## 性能前提

- 启用 Godot low processor mode。
- 活跃动画以 30 FPS 为上限，鼠标观察采样约 20～30 Hz。
- 静止后暂停不必要的骨骼、物理和持续 Shader 更新。
- 不使用依赖 `TIME` 的常驻效果维持待机动画。
- 不逐帧读取 Viewport 图片或重新计算精细透明轮廓。

首轮暂定性能门槛：

- 静止 30 秒后的平均 CPU 不高于当前机器单核的 3%。
- 普通眨眼、呼吸和视线组合的平均 CPU 不高于 12%。
- 8 小时待机后内存无持续线性增长，增长不超过 50 MB。
- 不出现身体比例、位置、窗口尺寸或命中轮廓的累积漂移。

这些数值会在首个技术样机测量后校正，但不能取消性能预算。

## 正式签名与公证

发布前需要：

1. 安装完整 Xcode并接受许可。
2. 加入 Apple Developer Program。
3. 配置 `Developer ID Application` 证书。
4. 主程序、GDExtension、动态库和辅助可执行文件全部构建为 Universal 2 并逐层签名。
5. 开启 Hardened Runtime，只申请实际需要的 entitlement。
6. 使用 `notarytool` 提交公证并 staple 票据。
7. 依次验证 `lipo`、`codesign`、`spctl` 和 `stapler`。

## 电脑控制组件前提

- 基础桌宠不申请 Accessibility 或 Screen Recording。
- 用户开启电脑控制后才下载固定 Bundle ID、固定 Team ID 的签名公证辅助组件。
- 下载后验证 SHA-256 与代码签名身份。
- 首次真正需要点击或键盘输入时才申请 Accessibility。
- 首次真正需要观察屏幕时才申请 Screen Recording。
- Apple Events 权限按具体应用、具体动作再申请。
- 辅助组件安装在用户级 Application Support，不申请 root，不常驻 LaunchDaemon。

主程序保管 API Key，并通过本地 IPC 向辅助组件发送经过权限判断的结构化任务；辅助组件不长期持有用户的 AI Key。

## 官方参考

- [Godot：Creating applications](https://docs.godotengine.org/en/4.6/tutorials/ui/creating_applications.html)
- [Godot：Window](https://docs.godotengine.org/en/4.6/classes/class_window.html)
- [Godot：DisplayServer](https://docs.godotengine.org/en/4.6/classes/class_displayserver.html)
- [Godot：Multiple resolutions / HiDPI](https://docs.godotengine.org/en/4.6/tutorials/rendering/multiple_resolutions.html)
- [Godot：Exporting for macOS](https://docs.godotengine.org/en/4.6/tutorials/export/exporting_for_macos.html)
- [Apple：Notarizing macOS software](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution)
- [Apple：Keychain Services](https://developer.apple.com/documentation/security/keychain-services)
- [Apple：SMAppService](https://developer.apple.com/documentation/servicemanagement/smappservice)
- [Apple：App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)
