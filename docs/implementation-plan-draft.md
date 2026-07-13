# 小纳雅桌宠重置版实施计划（讨论稿）

这份计划按依赖关系和可验证的纵向功能切分。角色渲染路线尚需用户确认专业工具预算，因此本文是可执行讨论稿，不是假装已经定案的最终排期。

## 架构决定

- 保留 Godot 4.6.3 作为角色、动画、窗口、气泡和管理 UI 的主程序。
- 角色渲染通过 `CharacterRenderer` 接口隔离，先比较 NativeRig 与 CubismRig。
- macOS 系统能力放入薄原生桥，不散落在角色脚本中。
- AI、上下文、长期记忆和电脑控制彼此分层。
- 基础桌宠不依赖 AI；电脑控制组件按需下载、默认关闭。

下文中的文件名是计划中的边界，不代表已经开始实现。单个任务控制在约 1～5 个核心文件，完成后立即验证。

## 阶段 0：环境与规则锁定

### 任务 0.1：建立可重复的 Godot 空工程

说明：锁定引擎、渲染器、最低系统和最小 CI，不加入角色功能。

验收条件：

- Godot 与导出模板均为 4.6.3 stable。
- Compatibility、low processor mode、macOS 13+ 和 Universal 2 写入配置。
- CI 能检查工程导入和脚本语法。

验证：无界面导入成功，最小工程可导出测试 App。

依赖：无。

计划文件：`project.godot`、`export_presets.cfg`、`.github/workflows/validate.yml`、`tests/smoke/import_project.gd`。

预计范围：M。

### 任务 0.2：定义并验证角色资源规范

说明：把分层、命名、画布、锚点和隐藏区域补画变成机器可检查的规则。

验收条件：

- 分层清单与命名规范获用户确认。
- 检查器能发现尺寸、透明边缘、缺层和锚点错误。
- 明确必须保存 PSD/PSB 和绑定源文件。

验证：正确样例通过，三个故意损坏的样例给出可读错误。

依赖：用户确认动画工具路线与制作方式。

计划文件：`docs/character-asset-spec.md`、`tools/validate_character_assets.py`、`tests/test_validate_character_assets.py`、`tests/fixtures/character-assets/manifest.json`。

预计范围：M。

### 检查点 0

- 环境可重复搭建。
- 资源规范获用户确认。
- 未开始批量绘制动作。

## 阶段 1：角色技术样机

### 任务 1.1：NativeRig 最小小样

说明：使用固定身体、刚性面部图层和少量 Godot 骨骼验证原生路线。

验收条件：

- 眨眼期间身体与衣服像素不变化。
- 视线、呼吸和辫子轻摆可以叠加。
- 一层测试服装切换后动作不中断。

验证：生成对齐差分报告、60 秒预览和性能采样。

依赖：0.2。

计划文件：`src/character/character_renderer.gd`、`src/character/native/native_rig.tscn`、`src/character/native/native_rig.gd`、`tests/character/native_rig_probe.gd`。

预计范围：M。

### 任务 1.2：CubismRig 最小小样

说明：只有用户接受 Live2D 路线时，才用固定版本 GDCubism 接入同一角色规范。

验收条件：

- Godot 4.6.3 编辑器和导出 App 均稳定启动。
- 眨眼、视线、呼吸、头发物理和服装参数互不覆盖。
- Universal 2 测试扩展可加载。

验证：与 NativeRig 使用同一预览、性能与 8 小时稳定性测试。

依赖：0.2；Live2D/绑定工具决定。

计划文件：`src/character/cubism/cubism_rig.tscn`、`src/character/cubism/cubism_rig.gd`、`vendor/gd_cubism.lock`、`tests/character/cubism_rig_probe.gd`。

预计范围：M。

### 任务 1.3：渲染路线对比与决策

说明：按相同条件比较，不凭宣传视频或单帧观感决定生产路线。

验收条件：

- 对视觉质量、制作效率、CPU、内存、稳定性、换装和许可证打分。
- 选定生产路线并写明回退方案。
- 未选路线能从主工程移除，不留下强依赖。

验证：用户观看同条件预览并明确批准。

依赖：1.1，以及适用时的 1.2。

计划文件：`docs/rig-comparison.md`、`docs/decisions/0001-character-renderer.md`。

预计范围：S。

### 检查点 1

- 角色可以自然眨眼且身体不抽动。
- 已有可继续编辑的母版和绑定源文件。
- 只有通过评审后才允许批量制作动作。

## 阶段 2：无 AI 的完整桌宠纵向切片

### 任务 2.1：透明角色窗闭环

说明：完成启动、拖动、命中轮廓、位置保存和重启恢复。

验收条件：

- 透明置顶角色窗不抢输入焦点。
- 简化命中轮廓准确，透明区域点击穿透。
- 处理多屏负坐标、Dock 和菜单栏可用区域。

验证：单屏、双屏、休眠唤醒、重启和插拔显示器测试。

依赖：检查点 1。

计划文件：`src/pet/pet_window.tscn`、`src/pet/pet_window.gd`、`src/platform/display_geometry.gd`、`tests/window/pet_window_probe.gd`。

预计范围：M。

### 任务 2.2：最小待机状态机

说明：实现慵懒默认姿势、眨眼、鼠标观察、夜间困倦和安全取消恢复。

验收条件：

- 每个动作定义进入、循环、退出、取消和恢复目标。
- 鼠标快速经过不追踪，停稳后才偶尔观察。
- 静止后进入低功耗模式，鼠标移动不持续唤醒全量渲染。

验证：状态转换测试、性能采样和 8 小时稳定性测试。

依赖：2.1。

计划文件：`src/behavior/idle_state_machine.gd`、`config/behaviors.json`、`config/animation_map.json`、`tests/behavior/idle_state_machine_probe.gd`。

预计范围：M。

### 任务 2.3：最小管理窗与窗口层级

说明：提供基本设置，并让管理窗、气泡、菜单和角色窗遵守统一优先级。

验收条件：

- 独立普通窗口提供显示、尺寸、置顶、开机启动和退出设置。
- 管理窗获得焦点时不被角色或气泡遮挡。
- 不配置 AI 时没有空白、报错或强制引导。

验证：同时操作角色窗、菜单和管理窗，焦点与层级稳定。

依赖：2.1。

计划文件：`src/ui/settings_window.tscn`、`src/ui/settings_window.gd`、`src/window/window_layer_coordinator.gd`、`tests/window/window_layer_probe.gd`。

预计范围：M。

### 检查点 2：第一个内部安装包

- 完全不接 AI 也能作为桌宠使用。
- 角色、动画和基础面板均为重置版。
- 只生成内部验证 App，不对外冒充正式 Release。

## 阶段 3：角色内容与界面完善

### 任务 3.1：首个待机动作包

说明：只加入半靠、观察鼠标和夜间困倦等 3～5 个可靠动作。

验收条件：动作衔接无跳位；大动作约 10～20 分钟调度；中断符合规则。

验证：逐动作循环预览、进入退出测试和随机调度回放。

依赖：2.2。

计划文件：`assets/character/actions/manifest.json`、`src/behavior/action_scheduler.gd`、`config/action_weights.json`、`tests/behavior/action_scheduler_probe.gd`。

预计范围：M。

### 任务 3.2：猴头菇精灵最小闭环

说明：先完成飞入、落肩和离开，抚摸动作留到这个闭环稳定后。

验收条件：精灵不是常驻；频率可配置；不与聊天、拖动或管理窗冲突。

验证：三种屏幕边缘、动作取消和重复出现测试。

依赖：3.1。

计划文件：`src/companion/spirit.tscn`、`src/companion/spirit_controller.gd`、`config/spirit_behaviors.json`、`tests/companion/spirit_probe.gd`。

预计范围：M。

### 任务 3.3：自适应气泡与视觉主题

说明：先完成最新消息气泡和统一主题，不在此任务加入聊天记录数据库。

验收条件：

- 气泡自动避开屏幕边缘和角色主体。
- 面板色彩、纹样和材质来自角色，而不是通用 AI 卡片模板。
- 角色移动和显示器切换时气泡不闪跳。

验证：屏幕四角、双屏和长短文本快照测试。

依赖：2.3。

计划文件：`src/ui/speech_bubble.tscn`、`src/ui/speech_bubble.gd`、`src/ui/naya_theme.gd`、`tests/ui/speech_bubble_probe.gd`。

预计范围：M。

### 检查点 3

- 新视觉语言、角色动作和桌宠节奏获得用户确认。
- 此后才开始 AI 与长期记忆功能。

## 阶段 4：AI 对话与本地记忆

### 任务 4.1：Provider 接口与 OpenAI 兼容纵向切片

说明：先打通 URL、Key、模型列表、流式回复和错误处理的一条完整路径。

验收条件：Key 进入 Keychain；伪 OpenAI 中转可配置；断网与错误响应可恢复。

验证：真实接口、模拟错误和无 Key 三种测试。

依赖：检查点 3；macOS Keychain 桥。

计划文件：`src/ai/provider_adapter.gd`、`src/ai/openai_compatible_adapter.gd`、`src/platform/macos_keychain.gd`、`src/ui/ai_settings_section.gd`、`tests/ai/openai_adapter_probe.gd`。

预计范围：M。

### 任务 4.2：Anthropic 与 Gemini 适配

说明：在 Provider 接口稳定后分别实现原生协议，不把差异塞进 OpenAI 适配器。

验收条件：模型刷新、流式消息、工具调用和 usage 统一归一化。

验证：Claude、Gemini 与不支持工具调用的降级测试。

依赖：4.1。

计划文件：`src/ai/anthropic_adapter.gd`、`src/ai/gemini_adapter.gd`、`src/ai/provider_capabilities.gd`、`tests/ai/provider_matrix_probe.gd`。

预计范围：M。

### 任务 4.3：白名单动作意图

说明：模型只输出结构化意图，由桌宠状态机决定是否执行。

验收条件：模型可选择表情、视线和白名单动作；无效参数和冲突动作安全拒绝。

验证：合法、越权、格式损坏和动作冲突测试。

依赖：4.1；2.2。

计划文件：`src/ai/action_intent.gd`、`src/ai/action_intent_router.gd`、`config/ai_action_schema.json`、`tests/ai/action_intent_probe.gd`。

预计范围：M。

### 任务 4.4：聊天记录与会话压缩

说明：先完成可恢复聊天记录、Token 预算和滚动摘要，不同时加入长期记忆编辑。

验收条件：历史本地保存；达到预算自动压缩；压缩完成立即继续；供应商原生压缩只作增强。

验证：长对话、重启恢复、压缩失败和中断恢复测试。

依赖：4.1。

计划文件：`src/context/context_engine.gd`、`src/context/chat_store.gd`、`src/context/compactor.gd`、`tests/context/compaction_probe.gd`。

预计范围：M。

### 任务 4.5：长期记忆与回收站

说明：加入 AI 可调用的长期记忆 CRUD、用户编辑、审计和软删除。

验收条件：固定人格只读；记忆可搜索、编辑、删除和恢复；所有变更有审计记录。

验证：跨会话召回、误删恢复、AI 越权删除和数据库迁移测试。

依赖：4.4。

计划文件：`src/memory/memory_store.gd`、`src/memory/memory_tools.gd`、`src/ui/memory_manager.gd`、`tests/memory/memory_store_probe.gd`。

预计范围：M。

### 检查点 4

- OpenAI 兼容、Claude 和 Gemini 三类接口通过兼容测试。
- 断网、接口错误、模型不支持工具调用时桌宠仍稳定。
- 重启后聊天与长期记忆可以恢复。

## 阶段 5：按需电脑控制

### 任务 5.1：能力包下载与供应链校验

说明：只实现下载、版本、签名、哈希和卸载，不执行电脑操作。

验收条件：仅在用户主动开启时下载；展示来源、体积和用途；校验失败不安装。

验证：正确包、篡改包、断点失败和卸载测试。

依赖：4.1；macOS 原生桥。

计划文件：`src/control/runtime_manager.gd`、`src/control/runtime_manifest.gd`、`src/ui/control_install_dialog.gd`、`tests/control/runtime_install_probe.gd`。

预计范围：M。

### 任务 5.2：默认只读权限闭环

说明：先接入文件、进程和浏览器观察，不允许任何修改。

验收条件：所有修改请求被拒绝并解释；操作记录可查看；关闭功能后辅助组件退出。

验证：只读成功、写入拦截、组件崩溃和紧急停止测试。

依赖：5.1。

计划文件：`src/control/control_client.gd`、`src/control/permission_policy.gd`、`protocol/control-ipc.schema.json`、`tests/control/read_only_probe.gd`。

预计范围：M。

### 任务 5.3：修改前审批

说明：实现第二档权限的说明、预览、批准、拒绝和超时。

验收条件：每次修改前显示对象、动作和影响；拒绝后不执行；批准记录可审计。

验证：文件修改、终端命令、超时和并发审批测试。

依赖：5.2。

计划文件：`src/control/approval_queue.gd`、`src/ui/control_approval_dialog.gd`、`tests/control/approval_probe.gd`。

预计范围：M。

### 任务 5.4：完全允许与持久化选择

说明：实现第三档权限，但保留审计、紧急停止和禁止自我修改边界。

验收条件：选择跨重启保持；用户可立即降级；紧急停止能终止正在运行的任务。

验证：重启、降级、停止和辅助组件失联测试。

依赖：5.3。

计划文件：`src/control/control_permission_store.gd`、`src/control/emergency_stop.gd`、`tests/control/full_access_probe.gd`。

预计范围：M。

### 检查点 5

- 主程序不长期向辅助组件暴露 API Key。
- Accessibility、Screen Recording 和 Apple Events 按需申请。
- 关闭功能后辅助组件停止运行并可完整卸载。

## 阶段 6：首个公开 Release

### 任务 6.1：签名、公证与 DMG 自动化

验收条件：主程序和所有内嵌组件逐层签名；Hardened Runtime 正确；公证和 staple 成功。

验证：`lipo`、`codesign`、`spctl`、`stapler` 全部通过。

依赖：完整 Xcode、Developer ID；检查点 4，电脑控制可延后到后续版本。

计划文件：`scripts/build_macos_release.sh`、`scripts/verify_macos_release.sh`、`.github/workflows/macos-release.yml`、`packaging/entitlements.plist`。

预计范围：M。

### 任务 6.2：发布验证矩阵

验收条件：Apple Silicon 实机、Intel 构建检查、Retina、外接屏、休眠唤醒、升级和卸载通过。

验证：保存版本化测试报告和性能基线。

依赖：6.1。

计划文件：`docs/release/verification-matrix.md`、`docs/release/performance-baseline.md`、`tests/release/smoke_check.gd`。

预计范围：S。

### 任务 6.3：GitHub Release 与回退

验收条件：Release 提供版本说明、SHA-256、已知限制和旧版本回退；安装包不进入 Git 历史。

验证：干净用户账号完成下载、安装、升级和卸载。

依赖：6.2。

计划文件：`docs/release/release-notes-template.md`、`scripts/publish_release.sh`。

预计范围：S。

## 主要风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| 没有可编辑绑定源文件 | 动画无法真正修复 | 把源文件交付列为第一阶段门槛 |
| GDCubism 在 Godot 4.6 不稳定 | 编辑器或导出崩溃 | 只做隔离小样，保留 NativeRig 退路 |
| 角色资产版权不明确 | 无法安全公开发行 | 代码和美术授权分开，正式发行前处理权利问题 |
| 待机持续高功耗 | 用户关闭或卸载 | 低功耗模式、30 FPS 上限、8 小时测量 |
| 多窗层级互相争抢 | 气泡和设置被遮挡 | 固定窗口角色优先级和统一协调器 |
| 功能同时开工导致返工 | 工程失控 | 严格按检查点推进，每阶段只交付纵向闭环 |
| 电脑控制权限过大 | 安全与信任风险 | 按需组件、默认只读、审批、审计、紧急停止 |

## 当前唯一关键决策

在批量制作角色资源前，必须先确定是否允许使用 Live2D Cubism 或 Spine 等专业绑定工具，以及是否可以在必要时请专业绑定师完成一次高质量绑定。这个答案会决定阶段 1 是否保留 CubismRig/Spine 小样，还是完全走 Godot NativeRig。
