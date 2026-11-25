![LocalAiNotif For HyperOS — 本地端大模型驱动的 HyperOS 智能通知模块](https://raw.githubusercontent.com/InternetBugs/LocalAiNotif-For-HyperOS/main/docs/images/banner-overview.png)

# 端侧AI通知总结 For HyperOS（LocalAiNotif For HyperOS）

> 本地端大模型驱动的 HyperOS 智能通知模块（LSPosed · 100% 离线端侧推理）

**LocalAiNotif For HyperOS** 是一款面向小米澎湃 OS（HyperOS）的开源 LSPosed 模块。模块内置 [MNN-LLM](https://github.com/alibaba/MNN) 端侧推理引擎，在设备本地运行 **Qwen3.5** 量化模型，按 **Hexagon NPU → Vulkan GPU → CPU** 自动选择并回退推理后端。模块为系统通知中心、锁屏、状态栏、音量面板与小米超级岛带来 **AI 通知摘要**、**单条长通知 AI 概括**、**重要通知专区**、**噪音过滤** 与 **减少干扰模式**。所有通知文本均在手机本地完成处理，**无需云端 API、不上传任何隐私数据、不依赖小米云控或小米账号**。

- 包名：`io.github.internetbugs.localainotif`
- 当前版本：**2.1.2**（versionCode `26`）
- [下载模块 APK](https://github.com/Xposed-Modules-Repo/io.github.internetbugs.localainotif/releases)
- [源码、构建说明与问题反馈](https://github.com/InternetBugs/LocalAiNotif-For-HyperOS)

---

## 环境要求与兼容性说明

> **重要提示**：当前实测环境为 **HyperOS 4 / Android 17**（主力机型：小米 15），**不保证更低版本系统或非 HyperOS 4 设备的兼容性**。

| 项目 | 要求 / 说明 |
|---|---|
| 测试环境 | **HyperOS 4 / Android 17**（小米 15），`minSdkVersion = 34` / `targetSdkVersion = 36` |
| 权限框架 | Root + **LSPosed**（`libxposed` API 102，静态作用域） |
| 芯片架构 | ARM64 (`arm64-v8a`)，原生支持 Android 15+ 16KB 页对齐 |
| 推理后端 | 自动按 **高通 Hexagon NPU**（内置 `v73 / v75 / v79 / v81 / v83 / v85` 六代 DSP 算子库，覆盖骁龙 8 Gen 2 / 8 Gen 3 / 8 至尊版 / 第五代骁龙 8 至尊版及新一代平台）→ **Vulkan GPU** → **CPU** 依次选择与回退。Vulkan GPU 加速取决于设备驱动，可用于骁龙、天玑、玄戒等平台。 |

> 模块深度对接 HyperOS 4 原生通知框架（`HyperNotificationManagerService`、`MiuiSystemUI`、音量面板插件、小米超级岛、`MIUINotificationCenter` 与系统设置）。若在未测试系统版本中遇到不匹配的 Hook 符号，对应特性会自动安全降级，不会导致系统崩溃，但部分视觉或交互能力可能不可用。

### 第三方模块兼容（HyperMusicCover）

**与 [HyperMusicCover](https://github.com/zyl6932/HyperMusicCover) 联用**（按 `v0.9.60` 适配）：两个模块同时在 LSPosed 的系统界面（`com.android.systemui`）作用域生效后，开启「减少干扰」时会联动收起锁屏超级岛内的普通通知。系统允许在锁屏展示的重要通知、来电与导航照常保留，在锁屏空白处上滑即可临时展开已静默的普通通知（自动避让媒体控制卡片区域）。

锁屏音乐卡片显示时，「减少干扰」标识会自动上移避让至卡片上方；音乐卡片收起后自动回到锁屏时钟下方。播放控制与封面手势仍交由音乐封面模块处理，基于 `MediaSession` 的自定义播放器通知也可正常放行；模式开启期间会自动隐藏日期旁的原生勿扰月牙图标。

---

## 核心功能

> **说明**：以下配图基于默认**「超级小爱流光」**主题绘制，实际渲染效果随所选光效风格（支持热切「星云极光」）、系统深浅色与壁纸材质实时自适应。

### 1. AI 摘要（折叠组多条汇总）
- **自然连贯第三人称摘要**：所有通知先按系统流程即时投递原文；同一应用的多条通知折叠成组后，由端侧模型在后台生成 Apple Intelligence 式 **1–2 句自然连贯的第三人称汇总**（保留发件人、关键动作及取件码/验证码/时间/金额/地点，保持原文语言）。
- **富样式上下文与确定性尾注**：完整提取 `MessagingStyle` 多轮聊天记录与 `InboxStyle` 多行列表，单次喂入最新 8 条并按时间正序整理；当组内通知超出喂入窗口或涉及多发件人时，由程序确定性补齐柔和余量尾注（如「等共 N 条通知」）。
- **宁缺毋滥护栏**：前置校验本地模型就绪状态（无模型零扫光、零请求），内置护栏拦截空输出、提示词泄漏、复读循环、照抄原文及丢失/篡改显式验证码或取件码，命中即静默保留原生折叠原文。
- **内联流光微标、自适应高度与 4 阶段动效**：不改动卡片底色与外框，折叠预览行首绘制自绘**双四角星芒 `✦✦ AI 摘要 |`** 与轻混流光正文，折叠卡片按实际排版高度自适应伸缩（上限 **4 行**），覆盖推理期扫光波、首次落地光浪洗字、亮屏重入徽标微光与折回轻闪。
- **灵活触发设置（二级页「摘要触发设置」）**：默认开启且对全部应用生效，支持即时调节**亮屏时生成摘要**、**合并等待时间**（`0 / 5 / 15 / 30 / 60 秒`）、**最小触发数量**（`1 / 2 / 3 / 5 条`）与**最小字数门槛**（`0 / 10 / 15 / 30 / 50 字`，重要通知不受限制），修改后无需重启立即同步。
- **无残留虚拟启用**：运行时在内存中接管 HyperOS 原生 AI 通知状态，不向系统写入 `ai_notification_setting`，停用或卸载模块后不污染系统设置。

### 2. AI 概括（单条长通知要点提炼）
- **面向单条长消息**：首页独立成节（默认关闭），开启后自动将单条较长通知提炼为 **1–2 句凝练要点**（优先呈现最新安排或结果及下一步行动，保留时间、地点与验证码），覆盖独立单条通知以及展开通知组后的长消息子卡；折叠组仍继续显示整组「AI 摘要」。
- **单星芒流光视觉与 4 阶段动效**：行首采用**单颗四角星芒 `✦ AI 概括 |`** + 渐变半粗体微标 + 渐隐竖光丝与轻混正文，严格维持单条通知原生行数、卡片高度与原生底色/外框。
- **原文无损与原生交互**：顶部弹出横幅保持原生原文不概括；锁屏是否展示遵循系统通知隐私设置；点开/展开该条通知即可完整阅读原文。
- **内联独立配置与即时重判**：常驻提供**亮屏时概括**与**触发字数门槛**（`50 / 80 / 120 / 200 字`），修改门槛后立即对通知栏现有通知重判生效。

### 3. 重要通知专区
- **纯端侧语义判定**：默认开启，由本地大模型实时识别**验证码、支付与交易、账号安全告警、物流异常、日程变更、健康告警、家人未接来电**等高优先级消息。
- **Apple 智能连续合卡 × 双光效环绕柔光**：在锁屏与通知中心自动置顶重要通知。通知中心内多条重要通知无缝合并为一张连续整卡并在最底端共用 7 叶交织丝带「重要通知」Footer；整卡外沿由 AGSL 着色器绘制柔和多色外扩散光晕与贴边玻璃折射棱。
- **解锁下拉刻意分层入场**：解锁态下拉通知中心且存在重要通知时，重要通知原生先行进入，下方普通通知平滑分层淡入微落。
- **独立 VIP 分组与专属摘要**：同一应用有 2 条及以上重要通知时，自动聚合为独立原生折叠组并仅基于组内 VIP 生成专属 AI 摘要。

![AI 摘要、AI 概括与重要通知专区示例](https://raw.githubusercontent.com/InternetBugs/LocalAiNotif-For-HyperOS/main/docs/images/demo-summary-priority.png)

### 4. 噪音过滤（广告推广收纳）
- **原样即时提醒，后台精准收纳**：首页独立开关（默认关闭）。通知到达时先按系统原设置即时提醒，随后由端侧模型在后台独立判定；未就绪模型或异常时保持原通知不变。
- **纯端侧语义自主判定**：以单条通知的主要用途为判据，由本地大模型结合提示词直接理解内容语义并独立分类，不叠加硬编码关键词/正则兜底；取件码、订单物流、账单、日程、聊天与新闻订阅即使附带优惠尾巴也按主要用途保留。
- **原生「推广通知」折叠页与无损恢复**：解锁后在通知中心底部沿用 HyperOS 原生「更多通知」样式展示独立「推广通知」入口与应用图标条，锁屏隐藏入口及已收纳广告；手动关闭开关时仍有效的已收纳通知会静默恢复至普通列表。

![噪音过滤与推广通知收纳示例](https://raw.githubusercontent.com/InternetBugs/LocalAiNotif-For-HyperOS/main/docs/images/demo-noise-filter.png)

### 5. 减少干扰模式（Reduce Interruptions）
- **智能分级勿扰**：默认关闭，复刻 iOS 18「减少干扰」体验，基于系统勿扰（`ZenMode`）构建并完整继承系统勿扰原生例外（来电、重复来电、闹钟、进行中事件）：
  - **普通与推广通知**：在通知中心静默保留、锁屏时默认收起；锁屏上滑一次即可临时展开查看并停留。
  - **重要通知穿透补发**：新通知到达后立即跳过摘要等待与字数门槛执行端侧分类，确认为重要通知后单次完整补发提醒（声震 / 横幅 / 亮屏）；无模型或超时时按 fail-closed 保持安静。
- **全系统多入口深度融合（7 叶交织丝带图标）**：覆盖**锁屏标识**、**控制中心互斥磁贴**、**音量面板第三行与原生定时**、**小米超级岛专属流光卡片**与**状态栏自适应图标**。
- **三项常驻精细开关**：支持**仅锁屏时生效**、**仅显示已放行通知的图标**与**保留应用内声音和系统音效**。

![减少干扰模式多入口融合示例](https://raw.githubusercontent.com/InternetBugs/LocalAiNotif-For-HyperOS/main/docs/images/demo-reduce-interruptions.png)

### 6. 双光效风格、本地模型与 HyperOS 原生设置
- **双「光效风格」无重启热切换**：支持在 **超级小爱流光（默认）** 与 **星云极光** 间一键热切，实时同步至通知栏全部 AI 视觉与设置页全部 5 组动态示意图。
- **三层异构推理与自动回退**：运行时按 **Hexagon NPU → Vulkan GPU → CPU** 自动探测与回退，所有后端共用同一份 MNN INT4 权重。
- **多镜像源一键下载与本地导入**：内置 **魔搭社区 ModelScope（默认推荐）**、**HF-Mirror** 与 **Hugging Face** 三下载源，支持断点续传、仅 Wi-Fi 下载保护与导入手机本地已有的 MNN-LLM 模型目录。
- **独立隔离进程与内存策略**：推理引擎隔离在独立 `:inference` 进程，默认空闲 **3 分钟** 自动卸载释放内存，同时提供**始终加载 AI 模型**开关。
- **HyperOS 原生设置与「关于」页**：支持**在系统设置中显示入口**（注入「通知与状态栏 › 小米超级岛」下方）、**隐藏桌面图标**、**应用内检查更新**与**运行环境诊断**。

![HyperOS 原生设置页与本地模型管理示例](https://raw.githubusercontent.com/InternetBugs/LocalAiNotif-For-HyperOS/main/docs/images/demo-settings-hyperos.png)

---

## 快速上手

### 第一步：安装与激活模块
1. 前往本仓库 [Releases](https://github.com/Xposed-Modules-Repo/io.github.internetbugs.localainotif/releases)（或通过 LSPosed 管理器仓库页）下载并安装最新正式版 `LocalAiNotif-v<version>.apk`（不含模型权重，体积轻巧；如需 Debug 调试包请前往主仓库 [Releases](https://github.com/InternetBugs/LocalAiNotif-For-HyperOS/releases)）。
2. 打开 **LSPosed 管理器**，启用 **端侧AI通知总结 For HyperOS（LocalAiNotif For HyperOS）**，确认以下 **5 个静态作用域**（见 [scope.list](https://github.com/InternetBugs/LocalAiNotif-For-HyperOS/blob/main/META-INF/xposed/scope.list)）已勾选：
   - **系统框架** (`system`)
   - **系统界面** (`com.android.systemui`)
   - **通知管理 / 通知中心** (`com.miui.notification`)
   - **设置** (`com.android.settings`)
   - **端侧AI通知总结 For HyperOS** (`io.github.internetbugs.localainotif`，用于「关于」页读取框架信息与激活状态)
3. **软重启或重启手机**使 Hook 生效（更新版本后若涉及系统 Hook 变更，软重启一次即可）。

### 第二步：下载或导入本地模型
1. 打开 **端侧AI通知总结 For HyperOS**，进入「本地模型 › 端侧模型」。
2. 选择下载源（国内网络推荐默认的**魔搭社区 ModelScope** 或 **HF-Mirror**）。
3. 下载预设模型 **Qwen3.5-0.8B（推荐）** 或 **Qwen3.5-2B** 并设为当前模型；也可以直接导入手机本地已有的 MNN-LLM 模型目录。

#### 预设端侧模型与实测资源占用

| 模型（MNN INT4） | 下载大小 | 推理后端 | 推理进程峰值内存 | 短通知摘要耗时 |
|---|---:|---|---:|---:|
| **Qwen3.5-0.8B（推荐）** | 约 550 MB | CPU | 924 MiB | 0.36 秒 |
| **Qwen3.5-0.8B（推荐）** | 同一份模型 | Vulkan GPU | 1,020 MiB | 0.68 秒 |
| **Qwen3.5-0.8B（推荐）** | 同一份模型 | Hexagon NPU + CPU | 299 MiB | 0.99 秒 |
| **Qwen3.5-2B** | 约 1.27 GB | CPU | 1,725 MiB | 0.73 秒 |
| **Qwen3.5-2B** | 同一份模型 | Vulkan GPU | 2,208 MiB | 2.72 秒 |
| **Qwen3.5-2B** | 同一份模型 | Hexagon NPU + CPU | 1,343 MiB | 4.99 秒 |

> 数据来自小米 15（骁龙 8 至尊版 `SM8750`）实测，每种后端独立运行三次取中位数。内置 Hexagon `v73 / v75 / v79 / v81 / v83 / v85` 六代 DSP 算子库，运行时通过 FastRPC 自动匹配；不可用时自动按 Hexagon NPU → Vulkan GPU → CPU 回退。

### 第三步：按需调整功能与入口
- **AI 摘要 / 重要通知**：默认开启并对全部应用生效，可在首页一键开关或进入「摘要触发设置」调整触发参数。
- **AI 概括**：默认关闭，开启后可随时调节「亮屏时概括」与「触发字数门槛（50 / 80 / 120 / 200 字）」。
- **噪音过滤**：默认关闭，开启后自动在后台将纯广告收纳至通知中心底部的「推广通知」。
- **减少干扰**：可在首页开启、点击「添加到控制中心」添加快捷磁贴，或在展开的 HyperOS 音量面板第三行直接开关并拖动定时。
- **光效风格 / 系统设置入口 / 隐藏桌面图标 / 检查更新**：在首页底部「其他」切换光效风格，或进入「关于」页按需配置入口与核验运行环境。

---

## 隐私与安全

- **100% 本地离线推理**：所有通知分类、折叠组摘要与单条概括均通过内置 MNN-LLM 引擎在设备本地完成，通知原文绝不离开本机。
- **极简按需联网**：仅在手动下载预设模型时访问所选模型仓库（ModelScope / HF-Mirror / Hugging Face），以及手动点击「检查更新 / 下载更新」时访问 GitHub；通知处理链路全程零联网。
- **进程与权限隔离**：模型权重与推理引擎隔离在独立 `:inference` 进程，系统核心进程 `system_server` 仅做轻量调度；`InferenceProvider` 按调用方 UID 严格校验推理与写配置权限。

---

## 常见问题（FAQ）

**Q：支持 HyperOS 2 / HyperOS 3 或 Android 15 / 16 吗？**  
A：当前仅在 **HyperOS 4 / Android 17（小米 15）** 完成端到端实测，**不保证更低版本系统或非 HyperOS 4 设备的兼容性**。部分深度 Hook 强依赖 HyperOS 4 组件签名，不匹配项会自动降级。

**Q：刚重启或首次收到通知时，为什么摘要、概括或重要通知识别需要等待一会儿？**  
A：默认按需加载模型并在空闲 **3 分钟** 无推理后自动卸载释放内存。如希望通知到达零等待，可在首页「本地模型」中开启**始终加载 AI 模型**。

**Q：「AI 摘要」和「AI 概括」有什么区别？**  
A：**AI 摘要（双星芒 `✦✦`）** 面向同一应用折叠起来的多条通知组；**AI 概括（单星芒 `✦`）** 面向单条长通知（含展开通知组后的长消息子卡），点开即可查看原文，两者互补不冲突。

**Q：未下载模型时开启「减少干扰」会怎样？**  
A：模型未就绪时按 fail-closed 策略保持安静，行为等同于传统系统勿扰模式（保留系统勿扰允许的来电、闹钟与进行中事件）；下载并启用模型后自动恢复 AI 重要通知穿透提醒。

**Q：为什么需要 `com.miui.notification`、`com.android.settings` 和 `io.github.internetbugs.localainotif` 作用域？**  
A：`com.miui.notification` 用于运行时虚拟保持系统 AI 通知状态并屏蔽小米云端下载弹窗；`com.android.settings` 用于原生设置支持与可选设置入口注入；`io.github.internetbugs.localainotif` 用于在「关于」页自检 LSPosed 框架版本与模块激活状态。

---

## 致谢与开源协议

- [MNN](https://github.com/alibaba/MNN) — 阿里开源高性能端侧推理引擎
- [Qwen3.5（0.8B / 2B）](https://huggingface.co/Qwen) — 通义千问端侧开源大模型（MNN 量化由 [taobao-mnn](https://huggingface.co/taobao-mnn) / [ModelScope MNN](https://www.modelscope.cn/organization/MNN) 提供）
- [LSPosed / libxposed](https://github.com/LSPosed/LSPosed) — 现代 Android ART Hook 框架

本仓库仅用于模块介绍与 APK 分发，不存放源码。完整源码、从源码构建说明与问题反馈请前往 [InternetBugs/LocalAiNotif-For-HyperOS](https://github.com/InternetBugs/LocalAiNotif-For-HyperOS)，采用 [GNU General Public License v3.0 (GPL-3.0)](LICENSE) 协议开源。
