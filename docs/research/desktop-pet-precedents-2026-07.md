# 桌宠前例与技术路线调研

调研日期：2026-07-13

## 结论先行

Godot 4.6 可以继续作为“小纳雅桌宠”的主程序。它已经具备透明窗口、置顶、多屏、2D 骨骼、动画状态机和完整 UI 能力。旧版失败的根因不是 Godot，而是使用了彼此不一致的整身 PNG 帧，并且没有保留可继续编辑的角色绑定源文件。

用户已经确认不购买 Live2D、Spine，也不聘请专业绑定师。因此 Live2D、Spine 只保留为调研对照，不进入必需依赖。Godot 作为桌宠主程序已经确定，但角色作者工具和实时运行时要分别通过技术门，不能过早锁死为纯手工 NativeRig。

重置版应采用：

```text
Godot 桌宠外壳
├── 窗口、多屏、气泡、面板与状态机
├── CharacterRenderer 角色渲染接口
│   ├── LayeredRig：固定分层实时眨眼、视线与嘴型
│   ├── BakedRig：同一绑定工程导出的复杂透明动作
│   └── ParametricRig：实时参数运行时通过技术门后再接入
├── 对话、上下文与本地记忆
├── macOS 原生桥：Keychain、开机启动、权限状态
└── 按需下载的电脑控制辅助组件
```

第一步不是制作几十个动作，也不是马上拆正式角色，而是用临时脸部测试件验证绑定编辑器、保存重开、透明导出和 Godot 运行路径。技术样机不过关，就不能扩大素材生产。

## 代表性前例

### VPet-Simulator

- 使用大量全身 PNG 序列帧，并建立了进入、循环、退出等动画分类。
- 后续不得不增加图集缓存、前置帧缓存和闲置资源清理。
- 实际 Issue 出现动作结束卡死、多屏边界错误、长期运行卡顿和内存升至约 1.2 GB。

可借鉴：动作生命周期和按需资源管理。

必须避免：把全部表现继续扩展为全身序列帧；它会让素材量、内存和一致性问题一起膨胀。

来源：[VPet](https://github.com/LorisYounger/VPet)、[动画卡死 #351](https://github.com/LorisYounger/VPet/issues/351)、[多屏 #518](https://github.com/LorisYounger/VPet/issues/518)、[内存 #530](https://github.com/LorisYounger/VPet/issues/530)。

### Shimeji / Shijima-Qt

- Shimeji 把动作定义与行为概率分开，这是正确方向。
- 传统角色包依赖固定编号的完整帧，启用多个角色后启动和内存成本明显增加。
- Shijima-Qt 为透明窗口、窗口追踪和跨平台行为使用了大量系统 Hack，最终因分支和维护成本归档。
- 真实问题包括透明点击遮罩变慢、多角色卡顿、多屏几何错误、Wayland 异常和 macOS 打包问题。

可借鉴：动作数据与调度逻辑分离；待机动作使用权重、冷却、时间和前置条件。

必须避免：认为“跨平台框架”会自动抹平系统差异。macOS 与 Windows 必须各有平台适配层。

来源：[Shimeji-Desktop](https://github.com/DalekCraft2/Shimeji-Desktop)、[Shijima-Qt](https://github.com/pixelomer/Shijima-Qt)、[性能 #55](https://github.com/pixelomer/Shijima-Qt/issues/55)、[多屏 #73](https://github.com/pixelomer/Shijima-Qt/issues/73)。

### PPet / Electron + Live2D 类桌宠

- PPet 使用 Electron、Pixi 与 Live2D，功能成熟但长期存在 CPU、内存、电量、多屏、透明背景和点击区域问题。
- Electron 不是不能使用，但对全天驻留的小型桌宠，需要额外对抗渲染器和后台动画功耗。
- Clawd on Desk 的真实测试显示，持续 SVG/CSS 动画可使空闲 CPU 达到约 20%；低功耗暂停后接近 0%。鼠标跟随一度持续唤醒渲染，修复后平均 CPU 从约 22% 降至 4.4%。

可借鉴：明确低功耗待机模式；鼠标观察属于被动行为，不能让整套渲染持续满速运行。

必须避免：关闭后台节流、永久播放动画、每次鼠标移动都全量重绘。

来源：[PPet](https://github.com/zenghongtu/PPet)、[PPet 性能 #157](https://github.com/zenghongtu/PPet/issues/157)、[Clawd 低功耗 #174](https://github.com/rullerzhou-afk/clawd-on-desk/issues/174)、[Clawd macOS CPU #244](https://github.com/rullerzhou-afk/clawd-on-desk/issues/244)。

### Dororo：最接近当前项目的 Godot 前例

- 使用 Godot、Live2D、透明置顶窗口、鼠标视线、触摸和 OpenAI 兼容聊天。
- 将帧率限制为 30 FPS，并启用低处理器模式。
- 为透明像素点击而逐帧把 Viewport 读取回 CPU，源码自己警告这会带来高成本和内存风险。
- 最重要的问题：作者只有导出的 Live2D 模型，没有绑定工程源文件，因此发现走路动画不自然后无法真正修复。

可借鉴：Godot 外壳与 Live2D 的组合确实可行。

必须避免：逐帧 GPU 回读；只保存导出模型而不保存 `.cmo3`、PSD 等可编辑源文件。

来源：[Dororo](https://github.com/MelanTech/Dororo)、[黑框 #8](https://github.com/MelanTech/Dororo/issues/8)、[缺少绑定源文件 #5](https://github.com/MelanTech/Dororo/issues/5)。

### Clawd on Desk：窗口与多屏的近期教训

- 多个置顶窗口如果各自反复恢复 `always on top`，会发生角色、气泡、HUD、菜单与设置窗口互相抢层级。
- macOS 的特殊窗口层级可能使透明命中窗覆盖输入气泡并吞掉点击，单纯降低普通窗口级别并不一定有效。
- 跨显示器保存尺寸时，如果反复读写经过 DPI 换算的实时窗口大小，可能出现每次休眠唤醒后逐渐变大。

对本项目的约束：角色窗、气泡和管理窗必须有固定的层级优先级；逻辑尺寸和像素尺寸不能互相回写；输入轮廓和视觉窗口分开测试。

来源：[窗口层级 #176](https://github.com/rullerzhou-afk/clawd-on-desk/issues/176)、[macOS 点击吞噬 #640](https://github.com/rullerzhou-afk/clawd-on-desk/issues/640)、[尺寸漂移 #408](https://github.com/rullerzhou-afk/clawd-on-desk/issues/408)。

## 角色动画候选路线

### A. Godot 原生分层骨骼

优点：

- 完全开源，无额外运行时许可。
- 与 `AnimationPlayer`、`AnimationTree`、脚本、声音和 UI 集成最直接。
- 可以精确控制固定身体、眼皮、瞳孔和饰品节点。

风险：

- Godot 的 `Polygon2D + Skeleton2D` 需要手工绘制网格、权重和内部顶点。
- 权重与内部三角形不合理时会产生折痕和橡皮感。
- 缺少成熟的专业角色绑定编辑体验，复杂面部与大幅全身动作制作成本较高。

适用方式：面部采用刚性分层与参数驱动；只对辫子、披肩和少量软组织使用骨骼或网格变形，避免把整张脸和身体强行蒙皮。

来源：[Godot 2D skeletons](https://docs.godotengine.org/en/4.6/tutorials/animation/2d_skeletons.html)。

### B. `nijigenerate MCP` + Godot

优点：

- BSD-2-Clause，免费开源，是基于 Inochi2D 技术继续发展的参数化 2D 角色编辑器。
- 内置 MCP 可直接导入 KRA/PSD、建立节点和网格、创建参数、写入逐顶点变形与 TRS 绑定、截图检查、保存和透明导出。
- Codex 可以通过结构化命令工作，不必把复杂绑定退化为坐标式 GUI 点击。

风险：

- MCP 目前只在 1.0 测试版提供，官方明确不保证稳定，必须操作副本并频繁另存。
- `nijilive/nicxlive` 没有现成 Godot 插件；实时接入需要单独开发和验证 GDExtension。
- 作者工具小样通过不等于运行时已经通过。

适用方式：先做临时脸部小样；第一版仍以 Godot 固定分层和同源烘焙序列作为安全运行基线。

来源：[nijigenerate](https://github.com/nijigenerate/nijigenerate)、[MCP API](https://github.com/nijigenerate/nijigenerate/blob/main/doc/mcp_api.md)、[nicxlive](https://github.com/nijigenerate/nicxlive)。详细评估见[免费角色绑定工具链调研](rigging-toolchain-options-2026-07.md)。

### C. Live2D Cubism + GDCubism

优点：

- 自动眨眼、呼吸、口型、参数混合、物理、表情和头发摆动体系成熟。
- 非常适合动漫人物的面部、眼神、呼吸和柔和摆动。
- GDCubism 支持 Godot 4.3+，声明支持 macOS Intel、Apple Silicon 和 Universal。

风险：

- GDCubism 是非官方扩展，没有可直接依赖的成熟二进制发行流程，需要自行构建并固定版本。
- 存在编辑器崩溃等开放 Issue，必须先在 Godot 4.6.3/macOS 真机验证。
- 眨眼、视线和换装能否稳定工作取决于模型参数规范；普通 Motion 可能覆盖服装参数，需要单独的参数持有策略。
- Live2D Framework 与 Core 使用不同许可证；个人或小规模主体发布固定内置模型的普通应用通常免 SDK 发布费，但允许导入不定数量模型的 Expandable Application 需要预审和特殊合同。

来源：[Cubism Native Framework](https://github.com/Live2D/CubismNativeFramework)、[GDCubism](https://github.com/MizunagiKB/gd_cubism)、[眨眼与视线 #17](https://github.com/MizunagiKB/gd_cubism/issues/17)、[换装参数 #91](https://github.com/MizunagiKB/gd_cubism/issues/91)、[崩溃 #164](https://github.com/MizunagiKB/gd_cubism/issues/164)。

### D. Spine Godot Runtime

优点：

- 官方提供 Godot 运行时、动作混合、Skin/Slot 换装、鼠标跟随与物理示例。
- 更擅长完整身体动作、道具和多套服装共用骨架。

风险：

- Spine Editor 是付费工具，运行时分发也受专门许可证约束。
- 导出数据版本必须与运行时版本严格匹配。
- 对动漫面部微表情的现成工作流通常不如 Live2D 直接。

来源：[spine-godot](https://github.com/EsotericSoftware/spine-runtimes/tree/4.2/spine-godot)、[Spine Runtime License](https://esotericsoftware.com/spine-runtimes-license)。

### 暂不采用

- 官方 Inochi2D Godot 扩展：当前 macOS + Godot 4.6 的 GDExtension 存在直接 abort 的开放问题；这不等同于排除独立发展的 `nijigenerate` 作者工具。
- Rive：运行时成熟，但没有官方 Godot 支持；现有 Godot 集成仍是小型 WIP 项目。
- 独立生成的全身 AI 动画帧：无法保证身份、锚点和身体轮廓稳定，明确淘汰。

来源：[Inochi2D #77](https://github.com/Inochi2D/inochi2d/issues/77)、[Rive Runtime](https://github.com/rive-app/rive-runtime)、[rive-godot](https://github.com/northernpaws/rive-godot)。

## 确定采用的技术验证顺序

1. 用临时脸部测试件验证 `nijigenerate MCP` 的导入、网格、参数、绑定、截图、保存重开和透明导出。
2. 用同一测试件建立 Godot 固定分层实时基线，并加载同源烘焙透明序列。
3. `nijigenerate` 小样通过后，再评估 `nicxlive -> Godot GDExtension` 的最小实时参数闭环。
4. 任一关键门失败，就转 Blender 自动化加 Godot NativeRig，不在正式角色素材上硬修测试工具。

必须通过以下验收表：

- 眨眼只改变眼部，身体像素与锚点不跳动。
- 瞳孔先看向鼠标，头部稍晚、幅度更小地跟随。
- 呼吸、眨眼、视线与辫子摆动可同时叠加。
- 一键切换服装层后，当前动作继续播放。
- 角色窗透明、可拖动，简化命中轮廓准确。
- 活跃动画限制在 30 FPS；进入静止后能降为低功耗模式。
- 连续运行 8 小时后无持续内存增长、比例漂移或锚点累积误差。

只有作者工具和至少一条 Godot 运行路径稳定后，才允许制作正式母版、增加姿势和大动作。Live2D、Spine 和当前崩溃的官方 Inochi2D Godot 扩展不作为隐藏依赖或临时捷径。

## 硬性避坑清单

- 不使用整身 AI 图片逐帧制作眨眼和微表情。
- 不把旧图中好看的局部抠到新图上拼接。
- 不只保留导出素材，必须保存分层母版和绑定工程。
- 不逐帧读取 GPU 纹理来判断透明点击区域。
- 不假设多屏坐标从 `(0,0)` 开始，也不把负坐标强制清零。
- 不让每个浮窗独立争抢置顶层级。
- 不让鼠标跟随阻止低功耗待机。
- 不允许动作任意打断；每个动作定义进入、循环、退出、取消与恢复状态。
- 不把设置、角色、AI、记忆和电脑控制继续塞进同一个脚本。
- 不把代码许可证与角色美术授权混为一谈。
