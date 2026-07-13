# 免费角色绑定工具链调研

调研日期：2026-07-13

## 结论先行

有比“在 Godot 里全靠手工绑”更合适的免费办法，但目前没有一个成熟项目能一键完成全部工作。

当前最科学的路线是：

```text
同一张获批角色母版
        ↓
KRA / PSD 统一分层与隐藏区域补画
        ↓
nijigenerate MCP 绑定小样（第一候选）
        ├── 通过：作为参数化作者工具
        └── 失败：Blender 自动化 + Godot NativeRig
        ↓
Godot 4.6.3 桌宠程序
        ├── 固定分层实时驱动：眨眼、瞳孔、眉毛、嘴型、轻呼吸
        ├── 同一绑定模型烘焙序列：复杂待机动作
        └── nicxlive GDExtension：通过独立技术门后才考虑实时接入
```

这条路线不购买软件，也不需要请绑定师；同时避免再次走回“每一帧让 AI 重画整个人，再抠局部拼接”的旧路。

## 候选方案对比

| 方案 | Codex 可自动化程度 | 运行时成熟度 | 质量上限 | 当前决定 |
|---|---:|---:|---:|---|
| `nijigenerate MCP` 作者工具 | 很高 | 需另接运行时 | 高 | 第一技术门 |
| Blender + Godot NativeRig | 很高 | 高 | 高，但工程量大 | 安全回退 |
| Godot 纯手工 NativeRig | 中 | 高 | 中高 | 只保留最小实时层 |
| Inochi Creator 官方线 | 低到中 | macOS Godot 当前不可用 | 高 | 仅离线参考/烘焙 |
| Live2D Cubism FREE | 低 | 第三方 Godot 插件有风险 | 高 | 免费对照，不做必需依赖 |
| Spine Professional | 中 | 有官方运行时 | 高 | 价格与许可证不符合要求 |
| Rive | 中 | 无官方 Godot Runtime | 中高 | 排除 |

## 1. `nijigenerate`：最接近“Codex 可以直接绑定”的开源工具

项目：[nijigenerate/nijigenerate](https://github.com/nijigenerate/nijigenerate)

- BSD-2-Clause，免费开源。
- 是基于 Inochi2D 0.8 技术继续发展的参数化 2D 角色编辑器。
- `v1.0.0-beta2` 提供 macOS DMG；当前 README 仍明确说明没有稳定构建，测试版可能意外崩溃。
- 内置本地 MCP Server，默认关闭，默认地址为 `127.0.0.1:8088/mcp`。

它的 MCP 不是模拟鼠标点界面，而是直接调用编辑器内部命令。已确认 Codex 可以：

- 导入 PSD、KRA、图片文件夹和 INP。
- 新建、移动、重命名、转换和删除角色节点。
- 自动生成网格，或直接写入顶点与三角形。
- 创建一维、二维和嘴部参数，并设置关键点范围。
- 为指定参数写入逐顶点 deform、平移、旋转和缩放绑定。
- 读回节点、参数和绑定数据。
- 截取当前视口，并给指定节点叠加边界或网格调试图。
- Undo/Redo、保存工程、导出 INP、透明 PNG、TGA 和视频。

这意味着可以建立“写入绑定 -> 截图检查 -> 读回数据 -> 失败撤销 -> 保存版本”的可验证闭环。

关键资料：

- [MCP API](https://github.com/nijigenerate/nijigenerate/blob/main/doc/mcp_api.md)
- [命令矩阵](https://github.com/nijigenerate/nijigenerate/blob/main/doc/commands.md)
- [v1.0.0-beta2](https://github.com/nijigenerate/nijigenerate/releases/tag/v1.0.0-beta2)
- [macOS nightly](https://github.com/nijigenerate/nijigenerate/releases/tag/nightly)

### 风险

- MCP 从 `v1.0.0-beta1` 才出现，生产覆盖面还小。
- GUI 必须保持运行，不是真正的无界面批处理器。
- MCP 默认无鉴权，只能绑定本机回环地址，不能暴露到局域网或公网。
- 曾出现 MCP DefineGrid 和 AutoMesh 保存崩溃，相关问题已修复；仍有旧的随机崩溃问题未关闭。
- macOS 安装包未见完整 Developer ID 签名和公证流水线，首次打开可能被 Gatekeeper 阻止。
- 不能让同一个工程在旧 Inochi Creator 与 nijigenerate 之间反复覆盖保存。

相关问题：

- [MCP DefineGrid 崩溃，已关闭 #169](https://github.com/nijigenerate/nijigenerate/issues/169)
- [AutoMesh 保存崩溃，已关闭 #168](https://github.com/nijigenerate/nijigenerate/issues/168)
- [随机崩溃，仍开放 #23](https://github.com/nijigenerate/nijigenerate/issues/23)

因此建议先测试 `v1.0.0-beta2`，全程只操作临时副本；只有遇到已经在主分支修复的问题时才测试 nightly。

## 2. Godot 中怎么运行角色

`nijigenerate` 解决的是“怎么制作参数化角色”，不自动解决“Godot 怎么实时渲染”。当前有三档方案。

### A. 固定分层实时驱动：第一版安全基线

Godot 直接控制独立的眼皮、瞳孔、眉毛、嘴型、脸底和服装节点：

- 眨眼只改变眼皮。
- 视线只移动瞳孔，头部稍晚且幅度更小地跟随。
- 辫子、羽毛和披肩只使用少量 `Skeleton2D` 或经过测试的弹簧。
- 身体、衣服主体和画布锚点不参与眨眼。

这是运行风险最低的部分，应当无论最终使用什么作者工具都保留。

### B. 同一绑定工程烘焙透明动作：复杂动作的保守方案

复杂待机可以从同一个参数化工程导出透明序列。它与旧版 AI 帧不同：角色的节点、网格、光影和坐标都来自同一工程，不会每帧重新生成一个不同的人。

代价是内存和动作组合自由度较低，所以只用于少量长待机、精灵互动或特殊镜头，不承担实时眨眼和鼠标视线。

### C. `nicxlive -> Godot GDExtension`：高自由度但高成本

项目：[nijigenerate/nicxlive](https://github.com/nijigenerate/nicxlive)

- BSD-2-Clause，C++20。
- 已实现大量 nijilive 节点、网格、遮罩、参数、物理和渲染兼容能力。
- 更适合未来封装为 Godot GDExtension。

但它目前没有现成 Godot 插件、没有稳定发布，原生 macOS 发布流水线和部分动画兼容仍不完整。真正接入还要自己处理纹理、顶点、UV、遮罩、混合模式、动态合成以及 macOS/Windows 打包。这是中型渲染集成，不是一个简单适配脚本。

所以它只能在最小模型加载、透明渲染、参数驱动、性能和长时间稳定性全部通过后进入正式工程。

官方 Inochi2D 的 Godot 扩展目前不能替代它：在 Godot 4.6.3/macOS 给节点赋 puppet 会直接 abort，问题仍开放：[Inochi2D #77](https://github.com/Inochi2D/inochi2d/issues/77)。

## 3. Blender + Godot：最稳的长期回退

[Blender](https://www.blender.org/) 免费开源，Apple Silicon 可用，且 `bpy` Python API 能让 Codex 稳定地脚本化：

- 批量建立分层平面与网格。
- 创建骨架、权重、约束和 Shape Keys。
- 创建眼皮、嘴型、头发与服装动作。
- 无界面渲染透明 PNG 序列。
- 导出带骨骼、蒙皮、形态键和动画的 glTF，或生成 Godot 原生资源。

参考：[Armature](https://docs.blender.org/manual/en/latest/modeling/modifiers/deform/armature.html)、[Shape Keys](https://docs.blender.org/manual/en/latest/animation/shape_keys/introduction.html)、[Python API](https://docs.blender.org/api/current/info_overview.html)、[glTF 导出](https://docs.blender.org/manual/en/latest/addons/import_export/scene_gltf2.html)。

### 可借鉴的 GitHub 项目

- [Proscenio](https://github.com/firebound/proscenio)：尝试把 Photoshop/Blender 角色导成 Godot 原生 `Skeleton2D`、`Polygon2D` 和 `AnimationPlayer`。GPL-3.0、项目很新、暂无成熟发布、当前依赖 Photoshop，适合隔离测试或借鉴导出格式，不能直接押宝。
- [Godot Skeleton2D Helper](https://github.com/folt-a/godot-skeleton2d-helper)：辅助建立骨骼、Polygon2D 与关键帧；文档和版本覆盖有限，只适合参考。
- [Souperior 2D Skeleton Modifications](https://github.com/ZedManul/souperior-2d-skeleton-modifications)：提供 LookAt、Jiggle 和 IK 思路；没有正式 Release，修改器顺序也可能产生抖动，应锁定提交并单独审计。
- [GD PSD Layer Import](https://github.com/Delsin-Yu/GD-PSD-Layer-Import-Plugin)：可自动把 PSD 图层导入 Godot；不支持所有蒙版、混合模式与半透明边界，不能代替母版检查。

不采用现成的所谓 Godot MCP 绑定项目作为生产依赖：目前发现的项目星数、测试覆盖和权重实现都不足，有的代码甚至没有真正把 Polygon2D 权重写入成功。需要自动化时，应围绕我们自己的资源规范编写小而可测的工具。

## 4. Codex 原生插件能帮到哪里

当前没有一个 Codex 原生插件能一键完成专业角色绑定。

- `MCXML Image2`：适合用多张原图锁定身份、生成同一母版候选、补画隐藏区域；不能生成可信的骨骼、权重和逐帧动画。
- 图像资源检查能力：适合检查透明边缘、锚点、尺寸跳动和眨眼差分。
- 电脑操作与录制回放：可以辅助操作没有 API 的编辑器，但复杂绑定界面容易退化为坐标点击。
- `nijigenerate MCP`：目前唯一找到的、能让 Codex 通过结构化 API 直接操作专业 2D 参数绑定器的外部开源方案。

因此绘图模型只负责“母版和缺失素材”，动画必须由绑定参数、骨骼、网格或同一绑定工程导出，不能再让绘图模型承担逐帧动画。

## 5. 官方付费工具价格与合法低价方式

用户已经决定不购买，以下价格只用于说明机会成本。人民币为页面参考汇率附近的约数，最终以结算页、税费和银行卡汇率为准。

### Live2D Cubism

- FREE：0 元，永久免费；个人或年销售额低于 1000 万日元的小规模主体可商用，但限制 1 张最高 2048px 纹理、100 ArtMesh、30 参数、50 Deformer、30 Parts 和 3 个 Blend Shape 参数。
- Indie 月付：2,080 日元/月，约人民币 87 元。
- Indie 年付首年：14,280 日元，约人民币 597 元。
- 学生/教师：7,344 日元/3 年，约人民币 307 元；当前 80% OFF 活动标注到 2027-03-31。
- 42 天 PRO Trial：免费，但试用期输出只允许内部或非商业用途，不能拿来做公开商业成品。

来源：[官方商店](https://store.live2d.com/en/)、[FREE/PRO 比较](https://www.live2d.com/en/cubism/comparison/)、[SDK 许可](https://www.live2d.com/en/sdk/license/)、[Expandable Application](https://www.live2d.com/en/sdk/license/expandable/)。

最便宜的合法路线是 FREE；若符合资格，学生三年优惠最划算。固定内置一个角色和允许用户无限导入模型的许可性质不同，后者可能属于 Expandable Application，需要预审和特殊合同。

### Spine

- Trial：免费，但不能保存或导出。
- Essential：官网当前显示 69 美元，约人民币 468 元；没有 Mesh，不足以制作本项目需要的高质量变形。
- Professional：官网当前显示 379 美元，约人民币 2569 元；才包含所需 Mesh、权重和完整功能。

官方明确没有学生优惠或经销商计划。来源：[Spine 官方购买页](https://esotericsoftware.com/spine-purchase)。

### Rive

- Free 不能导出给 Runtime 使用的 `.riv`。
- Cadet 年付折算 9 美元/月，或月付 17 美元。
- Runtime 是 MIT，但官方没有 Godot Runtime，社区插件仍是 WIP。

来源：[Rive 官方价格](https://rive.app/pricing)、[Runtime 导出说明](https://rive.app/docs/editor/exporting/exporting-for-runtime)。

不建议购买闲鱼共享账号、破解包或来路不明激活码：它们通常违反许可，也会给公开 GitHub 项目带来恶意软件、账号收回和无法更新的风险。

## 6. 第一轮验证方案

第一轮不使用正式纳雅，也不安装一堆插件：

1. 在隔离目录准备一个只有脸、双眼、眼皮、瞳孔和前发的固定尺寸 PNG 图层文件夹；不需要先安装 Krita。
2. 安装 `nijigenerate v1.0.0-beta2`，开启仅本机可访问的 MCP。
3. 让 Codex 自动导入、命名节点、建网格、创建 `EyeOpen` 与 `EyeXY` 参数并写绑定。
4. 自动截取开眼、半闭、闭眼和五个视线方向，连同网格覆盖图检查。
5. 保存、退出、重开，读回数据并再次截图。
6. 导出 `.inx`、`.inp` 和透明 PNG；连续重复三轮。
7. 再用 Godot 做一个固定分层实时眨眼基线，并评估同源透明序列的加载与功耗。

通过后才开始正式角色分层；失败则删除隔离样机，转 Blender/Godot NativeRig，不会污染主工程。

## 法律边界

上述工具的开源许可证只解决软件工具和代码，不会自动授予“纳雅”角色、美术和影视素材的版权。公开仓库应继续把程序代码与原作角色素材的授权问题分开说明，正式发行前需要单独评估。
