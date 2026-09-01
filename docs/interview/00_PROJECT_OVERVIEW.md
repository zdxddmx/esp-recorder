# 00 · 项目总览（面试开场用）

> 用途：面试自我介绍与「项目是做什么的」这一类问题。全文证据来自仓库源码 / 文档 / 提交记录，标注了证据位置。

## 一、一句话定义

> 这是一台基于 **ESP32-S3 + ESP-IDF（C / FreeRTOS）** 的**双路串口 WiFi 数据记录仪**（串口"黑匣子"）：接在两台串口设备之间，把两路 UART 收到的**原始字节流**实时保存到 SD 卡，同时通过 **AP+Station 双模 WiFi** 提供网页端实时监控 / 启停记录 / 文件管理，并通过 **USB CDC/MSC 复合设备**（虚拟串口 + U 盘读卡器）提供一条 USB 线的调试与取数通道。

- 证据：`README.md` L3-L5；`docs/ESP_Recorder_项目开发完整报告.md` L31-L40；`firmware/esp_recorder_v1.2/main/app_main.c` 整体初始化流程。

## 二、背景（面向客户定制）

- 这是一个**面向客户（维特智能）定制的产品**，不是通用开源玩具。项目负责人：**陈英炜**。
  - 证据：`README.md` L3、`docs/ESP32.md` L79-L86（"客户公司 维特智能 / 项目负责人 陈英炜 / 数量 100 / 交期 2026/05/31"）。
- **本人分工（【作者实习记录】`研发工作全景总结_20260709-0804_尤译庆.md` §2.3）**：
  - 本人（尤译庆）= **双通道数据接入、数据路由、SD 卡落盘、文件管理 API、参数持久化**；
  - 同事（蔡茹锦）= **Web 实时监控**。
  - 说明：`docs/ESP32.md` L85 的"项目组成员 陈英炜、游佳明"是客户侧需求文档记录，与作者自述的固件分工（尤译庆 + 蔡茹锦）口径不同，面试以作者自述为准，标【待本人确认】。
- 硬件形态是"非传统串口记录仪，无标准串口接口"，对外只有 **USB 口**（配网 + 底层通信）。
  - 证据：`docs/ESP32.md` L19-L25。
- 候选人的角色是**嵌入式软件执行者**（实习项目），报告作者署名"尤译庆"。
  - 证据：`docs/ESP_Recorder_项目开发完整报告.md` L816 "作者: 尤译庆"；`scripts/build_tech_report.py` L1030 / `build_report_docx.py` L683 均写 "作者：尤译庆"。

## 三、需求（从 docs 取证）

需求来自 `docs/ESP32.md`（产品功能描述）+ `docs/=任务.txt`（开发任务书）。核心可交付项（√ 为需求文档标注"已实现"的项）：

| 需求 | 来源 | 状态（仓库证据） |
|---|---|---|
| WiFi **AP+Station 混合模式**，AP 供手机直连、STA 连外网 | `docs/ESP32.md` L28-L31 | ✅ 代码 `wifi_config.c` `esp_wifi_set_mode(WIFI_MODE_APSTA)` |
| 网页端配置 + 数据监控 + 双向透传 | `docs/ESP32.md` L39-L47 | ✅ `web_server.c` 22 个 API + 内嵌 `index.html`（v1.2） |
| 支持开始/停止 SD 卡记录（局域网控制） | `docs/ESP32.md` L50-L53 | ✅ `POST /api/recorder/start` / `stop` |
| 支持调整串口波特率（适配未知波特率传感器） | `docs/ESP32.md` L56-L58 | ✅ v1.2 `user_uart_configure_channel()` 支持波特率/数据位/停止位/校验位 |
| **U 盘功能**：无网络时 Type-C 模拟 U 盘拷贝文件 | `docs/ESP32.md` L60-L61 | ✅ TinyUSB MSC（`usb_msc.c`），screenshot `V1.2_USB_CDC_MSC_硬件验证成功.png` |
| 后台服务器 / 远程 OTA / 固件升级 | `docs/ESP32.md` L64-L74 | ⚠️ 仓库中**未找到实现**（`docs/ESP_Recorder_项目开发完整报告.md` L738 自述"无 MQTT/云平台，仅局域网"） |

> `docs/=任务.txt` 是给 AI 开发执行者的任务书（要求组件化、`idf.py build`、网页预览、curl 验证、WebSocket、REST API、参数校验/路径穿越防护等）。它要求了 **WebSocket /ws**，但 v1.2 实际实现是 **200ms 增量轮询**（`web_server.c` 无 WebSocket 路由，`data_recorder.c` `recorder_live_fetch` 提供 since 增量拉取）——这一点面试时值得主动说明"为什么用轮询而非 WebSocket"。

## 四、难点（技术难点）

1. **USB CDC/MSC 复合设备 + SD 卡所有权互斥**：同一张 SD 卡，FATFS 本地记录 与 USB MSC 交给电脑当 U 盘不能同时访问。方案是 USB 挂载时 `recorder_stop()` + `sd_card_unmount_fs()`（只卸载 FATFS/VFS，保留块设备给 MSC），拔出后 `sd_card_remount_fs()` 重挂载恢复。证据：`app_main.c` L32-L51、`sd_card.c` L123-L167。
2. **高速 UART 数据落盘的"攒批 + 兜底"**：避免每个字节写 SD，用 16KB 环形缓冲 + 双条件（≥2KB 或 1 秒超时）+ 每秒 fsync。证据：`data_recorder.c` L21-L23、L308-L366。
3. **WiFi AP 永远在线 + STA 断线自动重连**：AP+STA 混合，STA 掉线不影响 AP。证据：`wifi_config.c` L37-L78。
4. **前后端 API 对齐 + 路径穿越防护**：v1.1 早期存在前端调用不存在的后端路由（`HANDOVER_AUDIT_STAGE1.md` L40），v1.2 补齐 `/api/wifi/scan`、`/api/config/uart` 等路由。证据：`web_server.c` L776-L797。

## 五、「我的工作内容」三档

> 分工口径（【作者实习记录】）：**本人 = 双通道数据接入 / 数据路由 / SD 卡落盘 / 文件管理 API / 参数持久化**；Web 实时监控主要由同事负责。讲"我"时聚焦前者，Web 部分用"我们/项目里"表述。

面试时按对方追问深度，选一档讲：

- **30 秒档（一句话）**：我用 ESP32-S3 + ESP-IDF 负责了这台双路串口 WiFi 数据记录仪的**数据链路侧**——双路 UART 接入、双向路由、SD 卡落盘、文件管理 API、参数持久化；设备插上就是 U 盘 + 虚拟串口复合设备，手机连热点能看数据、控记录。
- **3 分钟档（讲清架构）**：按"数据流"讲——UART 采集 → 环形缓冲 → 双条件落盘 SD + 实时流到 Web；外设讲 UART×2 / SD(SPI) / USB(CDC+MSC) / WiFi(AP+STA)；并发讲 FreeRTOS 5 个任务与互斥量。Web 实时监控模块说明由同事实现，我负责其背后的数据供给（live 环形缓冲 + `/api/stream`）。
- **8~10 分钟档（讲版本演进，最出彩）**：从 v1.1（双 UART + 透传 + SD + Web）→ v1.2（USB CDC/MSC 复合设备 + 完整 UART 参数 + NVS 配置 + 容错加固）→ v1.2_dev（usb_cdc_bridge）→ v1.3_release_test（cdc 测试协议 / 时间同步 / 安全测试模式）。详见 `11_INTERVIEW_SCRIPT.md`。

## 六、最终成果（以仓库证据 + 作者实习记录为准）

**能确凿证明的：**
- ✅ **编译通过**：ESP-IDF v5.4 + 目标 esp32s3，`idf.py build` 成功，产物 `esp_recorder_v1.0.bin`。证据：`docs/ESP_Recorder_项目开发完整报告.md` L714-L717；`firmware/esp_recorder_v1.1/artifacts/*.txt` 多份构建日志；`firmware/esp_recorder_v1.1_full_features/FULL_FEATURES_README.md` L37（SHA-256 固化产物）。
- ✅ **16 项功能模块代码完整**（`docs/ESP_Recorder_项目开发完整报告.md` L79-L97 功能清单，逐项标注"✅ 编译通过"）。
- ✅ **USB CDC/MSC 硬件验证有截图**：`screenshots/V1.2_USB_CDC_MSC_硬件验证成功.png`（文件名字面即"硬件验证成功"）。
- ✅ **部分实机验证通过（【作者实习记录】8.4 记录）**：SD 卡 SPI 挂载 ✓、WiFi AP 热点 ✓、Web 管理页面 ✓、数据记录引擎双条件落盘 + fsync 实机运行正常 ✓、USB 复合设备热插拔切换正常 ✓。
- ✅ **路由丢包实测（【作者实习记录】7.30 记录）**：CH2→CH1 路由 1 字节丢包 0 验证通过（CH1→CH2 当时因缺 USB-TTL 交叉接线暂未验证）。

**当时仍进行中 / 待确认的：**
- ⚠️ 作者 8.4 记录：**"Web 实时数据流推送 + 双路串口端到端验证持续中"** → 双路串口端到端最终状态标 **【待本人确认】**。
- ⚠️ 报告 12.2 的"待硬件验证"清单（`docs/ESP_Recorder_项目开发完整报告.md` L720-L727）与 `FULL_FEATURES_README.md` L52"没有执行烧录"、`HANDOVER_AUDIT_STAGE1.md` L6"未烧录开发板"**均为 8.3 及更早的表述**，被作者 7.29（固件烧录 ESP32-S3）、8.4（实机验证）的**更晚记录覆盖**。
- **结论**：面试措辞用"**编译通过 + 部分实机验证通过（SD/WiFi AP/Web/落盘+fsync/USB 热插拔）+ 双路串口端到端当时仍在收尾**"；**不要**说"全部功能实机验证通过"或"从未烧录"。

## 六·A 作者实习记录时间线（【作者实习记录】`研发工作全景总结_20260709-0804_尤译庆.md` §2.3）

| 日期 | 事实 |
|---|---|
| 7.24 | 项目启动 + 需求分析（数据采集→无线传输→记录存储流程审查） |
| 7.27 | 代码接手 + 分工：本人=双通道接入/数据路由/SD 落盘/文件管理 API/参数持久化；蔡茹锦=Web 实时监控 |
| 7.28 | 双路 UART 接入完成（CH1=UART0/CH2=UART2、Web 通道选择、REC_CH1/REC_CH2 双文件、字节/丢包统计）+ 修复三个问题（尾部缓存未落盘 / Web 输入 ID 冲突 / USB CDC/MSC 描述符冲突）+ ESP-IDF 5.4 编译通过 |
| 7.29 | 固件烧录 ESP32-S3：热点 `esp32wifi` + Web `192.168.10.1` 可访问；状态页新增计数器；"USB-TTL 数据未在 Web 实时区显示（排查中）" |
| 7.30 | Phase 1 双通道接入验证通过 + Phase 2 数据路由：CH2→CH1 1 字节丢包 0 验证通过；CH1→CH2 暂未验证（缺交叉接线） |
| 8.3 | 完整技术报告：14 源文件约 3600 行逐行审查、8 章 + 4 附录、16 步初始化、5 任务架构、双条件落盘、fsync、USB 热插拔、12 设计决策、14 REST API、5 排障 |
| 8.4 | 实机功能验证启动：SD SPI ✓ / WiFi AP ✓ / Web ✓ / 落盘+fsync ✓ / USB 热插拔 ✓；Web 实时流推送 + 双路串口端到端验证持续中 |

> 注：作者 7.29 实测热点为 `esp32wifi`/`192.168.10.1`，与最终代码默认 `ESP_Recorder_XXXX`/`192.168.4.1`（`wifi_config.c`）不同，属迭代中的早期烧录配置，面试可据实说明，不强行统一。

## 七、仓库级重要事实（写进每一份文档的前提）

1. **Git 历史只有 1 个 commit**：`43ae21f 整理并初始化 ESP32-S3 WiFi 数据记录仪项目`（2026-08-18）。**没有可用的 commit 级版本演进史**，版本演进只能靠 6 个 firmware 目录快照 + 目录内 artifacts 文件名日期重建。
2. **`firmware/esp_recorder_v1.2_dev` 与 `firmware/esp_recorder_v1.3_release_test` 是空 gitlink**（`git ls-files -s` 显示 mode `160000`，且 `.gitmodules` 不存在、子模块 commit 对象本地无法解析）。→ **v1.2_dev / v1.3 的源码不在本仓库**，对应"usb_cdc_bridge / cdc 测试协议 / 时间同步 / 安全测试模式"只能据 README 转述，无法逐行取证。
3. **`app/esp_recorder_app_v1.0` 同样为空 gitlink**，且 `docs/FLUTTER_ENV_SETUP.md` L127 自述"未创建 Flutter 项目"。→ Flutter App 源码**不在本仓库**。
