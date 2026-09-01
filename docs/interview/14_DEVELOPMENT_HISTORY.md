# 14 · 开发历史重建（git log + 版本目录 diff）

## 一、git 历史（真实，只有 1 个 commit）

```
commit 43ae21f2a1c00307d7c24db8e72f7c0c0197508f
Author: meminehobe24435-cmyk <meminehobe24435@gmail.com>
Date:   Tue Aug 18 17:31:06 2026 +0800
    整理并初始化 ESP32-S3 WiFi 数据记录仪项目
```

- **COMMITS = 1**，单分支 `main`，无 tag、无 merge、无历史提交。
- **含义**：这是"整理归档后的一次性初始化提交"，**丢失了增量演进史**。因此版本演进只能靠 **6 个 firmware 目录快照 + 目录内 artifacts 文件名日期**重建。
- ⚠️ Git 作者名 `meminehobe24435-cmyk`（GitHub 用户名）≠ 报告署名"尤译庆"。标【待本人确认】是否为同一人。

## 二、git 跟踪统计（证据）

- 总跟踪文件 **7975** 个。
- 按 firmware 版本分布：
  | 版本目录 | 跟踪文件数 |
  |---|---|
  | `esp_recorder_v1.1(1)` | 2010 |
  | `esp_recorder_v1.2` | 2012 |
  | `esp_recorder_v1.1` | 1959 |
  | `esp_recorder_v1.1_full_features` | 1956 |
  | `esp_recorder_v1.2_dev` | **1（空 gitlink）** |
  | `esp_recorder_v1.3_release_test` | **1（空 gitlink）** |

- `git ls-files -s` 显示 v1.2_dev / v1.3 / app 三个是 mode `160000`（gitlink，即嵌套 git 仓库），且 `.gitmodules` 不存在、子模块 commit 对象本地无法 `cat-file` 解析 → **内容丢失**。

## 二·A 作者实习记录时间线（【作者实习记录】`研发工作全景总结_20260709-0804_尤译庆.md` §2.3）

> 这是作者自述的第一手时间线，与仓库目录快照的 artifacts 日期可交叉印证。

| 日期 | 事件 |
|---|---|
| 7.24 | 项目启动 + 需求分析（审查 数据采集→无线传输→记录存储 流程） |
| 7.27 | 代码接手 + 分工：本人=双通道数据接入/数据路由/SD 卡落盘/文件管理 API/参数持久化；蔡茹锦=Web 实时监控 |
| 7.28 | 双路 UART 接入完成（CH1=UART0、CH2=UART2；Web 通道选择；REC_CH1/REC_CH2 双文件；字节/丢包统计）+ 修复三问题（尾部缓存未落盘 / Web 输入 ID 冲突 / USB CDC/MSC 描述符冲突）+ ESP-IDF 5.4 编译通过 |
| 7.29 | 固件烧录 ESP32-S3：热点 `esp32wifi` + Web `192.168.10.1` 可访问；状态页新增计数器；"USB-TTL 数据未在 Web 实时区显示（排查中）" |
| 7.30 | Phase 1 双通道接入验证通过 + Phase 2 数据路由：CH2→CH1 路由 1 字节丢包 0 验证通过；CH1→CH2 暂未验证（缺 USB-TTL 交叉接线） |
| 8.3 | 完整技术报告：14 源文件约 3600 行逐行审查、8 章 + 4 附录、16 步初始化、5 任务架构、双条件落盘、fsync、USB 热插拔、12 设计决策、14 REST API、5 排障 |
| 8.4 | 实机功能验证启动：SD SPI ✓ / WiFi AP ✓ / Web ✓ / 落盘+fsync ✓ / USB 热插拔 ✓；Web 实时流推送 + 双路串口端到端验证持续中 |

> 注意：7.29 实测热点 `esp32wifi`/`192.168.10.1` 与最终代码默认 `ESP_Recorder_XXXX`/`192.168.4.1` 不同（早期烧录配置 vs 最终默认），面试据实说明即可。

## 三、版本演进重建（按 artifacts 日期 + 文件差异取证）

> 时间顺序依据目录内 artifacts 文件名日期（v1.1 的 `stage1_dual_uart_build_20260729.txt`、v1.1(1) 的 `idf_build_20260727.txt`、v1.1_full_features 的 `full_features_build_20260803.txt`、报告文档日期 2026-08-03），现已与【作者实习记录】二·A 时间线交叉印证：7.27 接手审计（=v1.1(1)）→ 7.28/7.29/7.30 双 UART+路由（=v1.1）→ 8.3 报告+完整功能（=v1.1_full_features）→ v1.2 加固 → v1.2_dev/v1.3（源码缺失）。

### v1.1(1) —— 最早快照（单 UART + web 分离 + 接手审计）
- **证据**：无 `data_router.c`；有 `main/app.css`/`app.js` + `web/` 目录；`docs/HANDOVER_AUDIT_STAGE1.md`（2026-07-27）；artifacts `idf_build_20260727.txt`、`hardware_connection_validation_20260727.txt`。
- **一句话**：单路 UART + Web 分离文件的早期态，附带一份只读审计报告。

### v1.1 —— 双 UART 接入 + 数据路由
- **证据**：`main/data_router.c` 存在（CH1↔CH2 双向转发）；artifacts `stage1_dual_uart_build_20260729.txt`、`stage2_data_routing_build_20260730.txt`；`uart_driver.h` 定义 CH1/CH2 两路。
- **一句话**：**v1.1 打通"双路 UART 接入 + 双向透传 + SD 记录 + Web"全功能**（对应 `docs/ESP_Recorder_项目开发完整报告.md` 的 16 项功能清单）。

### v1.1_full_features —— 完整功能版（NVS 持久化 + 记录/文件互斥）
- **证据**：`FULL_FEATURES_README.md`（2026-08-03）列明"参数持久化到 NVS + SD config.json 镜像""记录期间禁止文件操作""USB 占用显示"；`config_manager.h` 新增 `router.enabled` 字段。
- **一句话**：在 v1.1 上补**配置持久化（NVS）+ 记录/文件操作互斥**。

### v1.2 —— 加固版（容错降级 + 完整 UART 参数 + 22 API + NVS 主存储）
- **证据**（v1.1→v1.2 关键 diff）：
  - `app_main.c`：初始化失败从 `ESP_ERROR_CHECK` 改 `ESP_LOGW` 降级（L92-L117）；
  - `uart_driver.c`：新增 `user_uart_configure_channel`（L163）+ `map_data_bits/stop_bits/parity`（L141-L161）；
  - `web_server.c`：`FW_VERSION="v1.2.0"`（L27），路由 14→22（新增 wifi/scan、wifi/disconnect、wifi/status、config/uart、config/router、config/recorder）；
  - `config_manager.c`：SD-only → **NVS 主 + SD 镜像**（L287-L330）；
  - 文件大小 diff：`web_server.c` 24.5K→31.9K、`config_manager.c` 9.9K→14.1K、`uart_driver.c` 7.0K→9.0K。
- **一句话**：**v1.2 把"能跑"变成"能扛"**——容错、补全、可配置。

### v1.2_dev —— USB CDC 桥接（源码缺失）
- **证据**：README L15/L26 称"新增 usb_cdc_bridge"；目录是空 gitlink。
- **一句话**：新增 `usb_cdc_bridge`（CDC↔UART 桥），**源码不在仓库，无法取证**【待本人确认】。

### v1.3_release_test —— 测试协议 / 时间同步 / 安全测试模式（源码缺失）
- **证据**：README L16/L26 称"含 cdc 测试协议、时间同步、安全测试模式"；目录是空 gitlink；`releases/esp_recorder_v1.3_release_test.zip`（README L33）走 GitHub Releases 分发、不入 git。
- **作者自述**（【作者实习记录】）：v1.3 含 cdc 测试协议 / 时间同步 / 安全测试模式，但**源码未入库**【待本人确认】。
- **一句话**：面向产测/交付的最终版，**源码不在仓库，无法取证**。

## 四、演进主线（一句话版）

```
v1.1(1)  单UART+Web分离+接手审计(07-27)
   ↓ 双UART+数据路由
v1.1     双UART接入+双向透传+SD+Web 全功能(07-29/30)
   ↓ NVS持久化+记录/文件互斥
v1.1_full_features  完整功能版(08-03)
   ↓ 容错降级+完整UART参数+22API+NVS主存储
v1.2     加固版
   ↓ usb_cdc_bridge (源码缺失)
v1.2_dev USB CDC 桥接
   ↓ cdc测试协议/时间同步/安全测试模式 (源码缺失)
v1.3_release_test  交付/产测版
```

## 五、审计要点（给候选人的 3 句话）

1. **唯一 commit 是"整理归档"**：面试被问 git 历史，如实说"仓库是归档后的单次初始化提交，增量历史不在里面，我按目录快照重建了演进"。
2. **v1.2_dev / v1.3 是空 gitlink**：这是仓库完整性缺口，提前主动声明，别等面试官点进去发现空目录。
3. **演进主线选 v1.1→v1.2 讲**（这两版源码齐全、diff 清晰、能逐行举证），v1.2_dev→v1.3 只讲目标、不吹实现。
