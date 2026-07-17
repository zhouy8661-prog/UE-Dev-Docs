**UE5 UI项目开发规范**

UI Development Guideline

------ 程序版 ------

适用范围：UE5 项目中所有 UMG 界面、UI资源、UI动效、UI材质、UI性能优化

版本：v1.0 \| 密级：内部

**1. UI资源导入与纹理设置**

**1.1 Texture导入设置**

所有 UI Texture 导入 UE5 后必须检查以下设置，这是 UI 渲染质量的基础保障。

**Compression Settings（压缩设置）**

> UserInterface2D (RGBA)

- 保留 Alpha（透明通道）

- 避免颜色失真

- 提升 UI 清晰度

**Texture Group（纹理分组）**

> UI

让 UE5 使用 UI 专用纹理管理策略，确保渲染优先级和内存策略正确。

**Mip Gen Settings（Mip生成设置）**

> NoMipmaps

UI 不存在远距离观察需求。关闭 Mipmap 可：

- 避免缩小时模糊

- 减少额外纹理数据（节省约 33% 显存）

**Filter（过滤方式）**

> Bi-linear 或 Tri-linear

用于平滑边缘、减少像素噪点。不推荐使用 Nearest（最近邻）过滤。

**2. UI资源命名规范**

**2.1 命名格式**

**统一格式：**

> 类型_模块_名称_用途

**2.2 Texture 命名**

> T_UI_模块_名称

- 登录背景：T_UI_Login_BG

- 技能图标：T_UI_Icon_Sword

- 按钮：T_UI_Button_Normal

- 背包图集：T_UI_Inventory_Atlas

**2.3 Material 命名**

> M_UI_功能

- 血条流动：M_UI_HPFlow

- 能量波动：M_UI_EnergyWave

**2.4 Material Instance 命名**

> MI_UI_功能_变体

- 红色血条：MI_UI_HPFlow_Red

- 蓝色能量：MI_UI_EnergyWave_Blue

**2.5 Widget Blueprint 命名**

> WBP_功能名称

- 主菜单：WBP_MainMenu

- 背包：WBP_Inventory

- 设置界面：WBP_Settings

**2.6 Material Function 命名**

> MF_UI_功能

- 能量流动函数：MF_UI_EnergyFlow

**3. UI性能优化规范**

**3.1 Overdraw优化**

Overdraw（过度绘制）指同一区域被 GPU 重复绘制。禁止大量透明空白区域。

**错误示例：**

> 1024×1024图片，实际内容仅 200×200

**优化方案：**

- 裁剪透明区域，仅保留有效像素区域

- 使用 UE5 Texture Editor 中的 Crop 功能

**3.2 Texture Atlas（纹理图集）**

同一界面的资源必须合并为图集，减少 Draw Call 和 Texture 切换。

**错误：**

> 20个装备图标 = 20张 Texture

**正确：**

> T_UI_Inventory_Atlas（1张 1024×1024）

- 显著减少 Draw Call（从 20+ 降至 1-3）

- 减少 GPU Texture 切换开销

**3.3 禁止 UI Tick**

严格禁止在 Widget Blueprint 中使用 Event Tick 修改 UI 属性（颜色、透明度、位置等）。

> ✖ Event Tick → 修改颜色 → 修改透明度

原因：每帧执行导致 CPU 消耗、蓝图性能下降。

**正确方案：**

> ✔ Time Node → Material Animation（GPU 执行）

- 动画交给 GPU 执行

- 使用 Material Parameter Collection 全局控制

**3.4 Draw Call 优化原则**

- 同一 Widget 内尽量使用同一张 Atlas

- 减少 Widget 层级嵌套深度

- 合理使用 Invalidation Box 减少重绘

- 避免动态更改 Widget 结构树

**4. UI蓝图架构规范**

**4.1 推荐架构**

推荐 UI 分层架构：

> UI Layer ├── Widget（纯显示层） ├── UI Controller（逻辑控制层） ├── Data Struct（数据层） └── Material Parameter（视觉参数层）

**4.2 职责分离**

**UI 负责：**

- 显示渲染

- 输入响应

- 动画触发

**UI 不负责：**

- 游戏逻辑

- 数据计算

- 状态管理

**4.3 数据驱动原则**

- 所有 UI 状态由 Data Struct 控制

- 通过 Event 通知 Widget 更新

- 使用 BindWidget 绑定子控件

- 避免 Widget 直接访问 GameMode/GameState

**5. Channel Packing规范**

**5.1 通道打包目的**

减少 Texture 数量、Draw Call 和 GPU 读取次数。

单张 RGBA Texture 存储多种数据：

  -------------- --------------------------------------------------------
  **通道**       **用途**

  R              Shape Mask（形状遮罩）

  G              Glow Mask（发光遮罩）

  B              Detail Noise（细节噪声）

  A              透明度
  -------------- --------------------------------------------------------

**5.2 优化效果**

原本需要：

> Mask1 + Mask2 + Noise = 3张 Texture

优化后：

> 1张 RGBA Texture

- 降低 GPU Bandwidth（带宽压力）

- 减少纹理采样次数

**6. UI Material开发规范**

**6.1 Material Domain设置**

**UI 材质必须设置：**

> Material Domain: User Interface

这确保材质在 UMG 中正确渲染，且使用 UI 专用渲染通路。

**6.2 动效实现原则**

以蓝色能量条为例，材质节点结构：

> Noise Texture ↓ Panner（UV移动节点） ↓ Multiply（混合） ↓ Mask控制范围 ↓ Final Color

所有可调属性全部使用 Parameter：

- 流动速度（FlowSpeed）

- 颜色（FlowColor）

- 亮度（GlowIntensity）

- 透明度（Opacity）

**6.3 参数命名规范**

示例：MF_UI_EnergyFlow

> Parameters: FlowSpeed GlowIntensity FlowColor MaskPower

这些参数方便程序通过 Blueprint 或 C++ 动态控制。

**6.4 Material Instance 使用**

- 每种颜色/风格变体创建 MI 而非新 M

- 共享同一母材质的节点逻辑

- 减少 Shader 编译数量

**7. UI动画开发规范**

**7.1 禁止方案**

**禁止视频播放动画**

> ✖ AE动画 → 导出MP4 → UE播放

原因：

- 视频无法动态交互

- 无法响应游戏状态

- GPU 占用高、内存占用大

- 包体增加显著

**禁止大量序列帧**

> ✖ AE序列帧 → 大量Texture → UE播放

原因：

- Draw Call 增加

- 显存压力巨大

- 包体增加

**7.2 推荐方案：Material Driven Animation**

**流程：**

> AE设计效果 ↓ 拆解动画元素 ↓ 输出基础素材 ↓ UE Material实现 ↓ 参数控制

**7.3 UMG Animation 使用规范**

- 适用于 UI 位移、缩放、流入流出等简单动画

- 复杂视觉效果始终使用 Material 实现

- 动画绑定到 Widget 生命周期事件（Construct/PreConstruct）

**8. UI交付检查表**

**8.1 UE导入检查**

- [ ] Compression: UserInterface2D

- [ ] Texture Group: UI

- [ ] Mipmap: NoMipmaps

- [ ] Filter: Bi-linear / Tri-linear

**8.2 命名规范检查**

- [ ] Texture 符合 T_UI_模块_名称 格式

- [ ] Material 符合 M_UI_功能 格式

- [ ] Widget 符合 WBP_功能名称 格式

- [ ] Material Instance 符合 MI_UI_功能_变体 格式

**8.3 性能检查**

- [ ] 无 Widget Event Tick 使用

- [ ] Draw Call 合理（单界面建议 \< 50）

- [ ] Texture Atlas 完成合并

- [ ] Overdraw 检查完成（使用 Console: r.VisualizeOverdraw 1）

- [ ] Channel Packing 实施完成

**8.4 动效检查**

- [ ] 无视频播放动画

- [ ] 无大量序列帧贴图

- [ ] 复杂动效使用 Material 实现

- [ ] 参数可动态调整

- [ ] Material Domain 设为 User Interface

**9. 核心原则与开发流程**

**9.1 核心原则**

  ------------------ ----------------------------------------------------
  **原则**           **说明**

  资源标准化         统一尺寸、命名、格式，降低沟通成本

  材质驱动           复杂动画交给 GPU 执行，减少 CPU 负担

  数据驱动           参数控制效果，解耦 UI 与逻辑

  减少绘制           优化 Draw Call，合理使用 Atlas

  避免Tick           减少 CPU 压力，动画 GPU 化

  模块化设计         方便扩展维护，提高复用率
  ------------------ ----------------------------------------------------

**9.2 开发流程**

> UI需求设计 ↓ UI交互方案 ↓ 美术输出规范资源 ↓ UE Texture规范导入 ↓ Material实现动态效果 ↓ Widget搭建 ↓ 性能Profiler检测 ↓ Release发布
