# 12 · GitHub 现场演示路线（5 分钟）

> 目标：让面试官在浏览器里顺着你的讲述看到真代码。每一步给出"点哪个文件 + 指哪几行 + 说哪句话"。

## 路线总览

| 步骤 | 时间 | 打开 | 指哪 | 说哪句话 |
|---|---|---|---|---|
| 0 | 0:00 | `README.md` | L3-L26 | 一句话定义 + 版本沿革 |
| 1 | 0:30 | `firmware/` 目录 | 6 个版本目录 | "先看版本演进，这是我项目的主线" |
| 2 | 1:00 | `firmware/esp_recorder_v1.2/main/app_main.c` | L71-L200 | 初始化顺序 + 容错降级 |
| 3 | 1:40 | `main/data_recorder.c` | L308-L366 | 双条件落盘 + fsync（最核心） |
| 4 | 2:20 | `main/sd_card.c` + `main/usb_cdc.c` | L123-L167 / L30-L57 | SD↔USB 所有权互斥 + 复合描述符 |
| 5 | 3:00 | `main/web_server.c` | L776-L797 | 22 个 API 路由 |
| 6 | 3:40 | `docs/ESP32.md` → `docs/ESP_Recorder_项目开发完整报告.md` | 需求 L28-L61 / 报告 L79-L97 | 需求→交付闭环 |
| 7 | 4:30 | `scripts/build_tech_report.py` | L16-L24 | 报告脚本（工程化交付） |

## 逐步话术

**步骤 0（README，30s）**
打开 `README.md`，指 L3-L5："面向维特智能客户定制，AP+Station、双向透传、SD 记录、USB CDC/MSC、Web 配置监控"。指 L24-L26："版本沿革 v1.1→v1.2→v1.2_dev→v1.3"。

**步骤 1（版本目录，30s）**
展开 `firmware/`，按顺序点 6 个目录名，一句话各是什么（见 `14_DEVELOPMENT_HISTORY.md`）。补一句："v1.2_dev 和 v1.3 的源码不在这个仓库里，是空 gitlink，我后面会讲。"

**步骤 2（app_main，40s）**
打开 `app_main.c`，滚到 L92-L94 指注释 "SD card not available — WiFi & Web will still work"："v1.2 我把初始化失败从硬 abort 改成了降级继续跑。"再指 L120-L135："两路 UART 独立初始化，一通道坏不影响另一通道。"

**步骤 3（data_recorder，40s）**
打开 `data_recorder.c`，指 L21-L23 三个宏："16KB 缓冲、2KB 阈值、1 秒超时，双条件落盘。"指 L334-L366："rec_sync 每秒 fsync，连续 3 次失败关文件。"

**步骤 4（sd + usb，40s）**
打开 `sd_card.c` L133-L150，指注释"只卸载 FATFS/VFS，保留 sdmmc_card_t 供 USB MSC"："这就是 USB 插拔时 SD 卡所有权的互斥切换。"切 `usb_cdc.c` L30-L41："CDC 接口 + MSC 接口并列在一个配置描述符里，复合设备就是这么枚举的。"

**步骤 5（web_server，40s）**
打开 `web_server.c`，滚到 L776-L797 路由表："22 个 API，v1.1 只有 14 个，v1.2 补上了前端真正调用的 wifi/scan、config/uart 这些。"

**步骤 6（需求→交付，50s）**
打开 `docs/ESP32.md` L28-L61："这是产品需求，AP+STA、透传、SD 记录、U 盘、波特率可调。"切 `docs/ESP_Recorder_项目开发完整报告.md` L79-L97："这是交付报告，16 项功能逐项打了编译通过。"

**步骤 7（报告脚本，30s）**
打开 `scripts/build_tech_report.py` L16-L24："我连交付报告都是用脚本从代码生成的，改代码后一键重出报告，内容和代码强绑定。"

## 收尾一句（留给提问）
"如果要深挖，我最想聊的是 USB CDC/MSC 和 SD 卡的互斥，以及 UART 高吞吐下的落盘背压——这两个是我认为最难、也最想继续优化的点。"

## 演示红线（别踩）

1. **不要**打开 `managed_components/`（第三方 TinyUSB，几千文件，且与主题无关）。
2. **不要**在没准备的情况下打开 `sdkconfig` 谈 Flash——先记住 2MB/16MB 的矛盾（`07_HARDWARE_INTERACTION.md` §六），否则被问会卡壳。
3. **不要**声称"v1.3 源码在这里"——v1.2_dev/v1.3 是空 gitlink，点进去是空的，提前主动说明反而加分。
