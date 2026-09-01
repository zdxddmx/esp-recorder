# 13 · 代码质量审计（只记录，不改代码）

> 审计范围：`firmware/esp_recorder_v1.2/main/*.c/*.h` + `sdkconfig` + `partitions.csv`。按维度给结论 + 证据 + 严重级。严重级：🔴 高 / 🟡 中 / 🟢 低。

## 一、缓冲区与边界

| # | 问题 | 证据 | 级别 |
|---|---|---|---|
| 1 | 环形缓冲 `ringbuf_write` 满时静默截断（写多少算多少，差额计 drop） | `data_recorder.c` L82-L93、L299-L303 | 🟡 有 drop 统计但无告警 |
| 2 | UART 读缓冲 256B 栈数组，读超时 100ms 轮询 | `app_main.c` L56-L60 | 🟢 可用，非最优 |
| 3 | `recorder_get_filename` 无锁读 `state->filename` | `data_recorder.c` L457-L470（对比其他 getter 都持 s_mutex） | 🟡 潜在 TOCTOU/半读 |
| 4 | live 行 `snprintf` 有边界保护（`pos + 3 < sizeof(line)`） | `data_recorder.c` L252 | 🟢 正确 |
| 5 | 字符串 copy 用 `strncpy(..., size-1)` + 数组初始化，未显式补 '\0' 但结构体整体 `{0}` 初始化 | `wifi_config.c` L152-L165 | 🟢 基本安全 |
| 6 | `config_manager` JSON 大小上限 4096，读文件 `ftell` 判 `size<=0 || >CONFIG_MAX_SIZE` | `config_manager.c` L227-L232 | 🟢 正确 |
| 7 | 路径穿越防护 `is_safe_filename` 拒绝 `/ \ : ..` | `web_server.c` L50-L58 | 🟢 正确（但依赖所有文件接口都调用它） |

## 二、SD / FATFS 错误处理

| # | 问题 | 证据 | 级别 |
|---|---|---|---|
| 8 | `fwrite` 部分写入 → 置 failed 但**不重试**，直接 break | `data_recorder.c` L188-L200 | 🟡 丢失即放弃 |
| 9 | `fopen` 失败 → 丢整批 + drop_count，不重试 | `data_recorder.c` L288-L297 | 🟡 |
| 10 | `fsync` 连续 3 次失败才关文件，关闭后**无自动重开/告警** | `data_recorder.c` L351-L362 | 🟡 静默降级 |
| 11 | SD 未挂载时 `recorder_flush_all_locked` 把缓冲数据计 drop 后丢弃 | `data_recorder.c` L208-L218 | 🟢 有意设计（SD 已被 USB 接管），但无日志 |
| 12 | SD 卡热插拔无检测（只靠 USB mount/unmount 事件） | `sd_card.c` 无 GPIO 检测 | 🟡 |
| 13 | `format_if_mount_failed=false`（不自动格式化） | `sd_card.c` L62 | 🟢 保护用户数据（正确取舍） |

## 三、WiFi 重连 / 网络

| # | 问题 | 证据 | 级别 |
|---|---|---|---|
| 14 | 事件回调内 `vTaskDelay(5000)` 阻塞默认事件循环 | `wifi_config.c` L44-L53 | 🔴 并发/阻塞风险 |
| 15 | 重连计数 `s_retry_count` 无锁（在事件循环单线程上下文 + handler 内访问，风险较低但非显式保护） | `wifi_config.c` L26、L49-L62 | 🟢 单事件循环内可接受 |
| 16 | AP 密码明文打印日志 `ESP_LOGI(...PASS=%s...)` | `wifi_config.c` L178-L180 | 🟡 日志泄漏（`HANDOVER_AUDIT` L39 也指出） |
| 17 | `wifi_connect_sta` 对 `ESP_ERR_WIFI_STATE` 用 stop→set→start 兜底，而非 abort | `wifi_config.c` L219-L227 | 🟢 良好 |
| 18 | 无 HTTPS / 无鉴权 | `web_server.c` L26 `WEB_PORT 80` | 🟡 局域网内可接受，公网不可 |

## 四、并发

| # | 问题 | 证据 | 级别 |
|---|---|---|---|
| 19 | `xSemaphoreGive(s_data_sem)` 在 `xSemaphoreGive(s_mutex)` 之后，且每次写都 give | `data_recorder.c` L304-L305 | 🟡 信号量计数膨胀（靠 1s 超时兜底） |
| 20 | UART 计数器、router 统计用 portMUX 保护 | `uart_driver.c` L211-L219、`data_router.c` L49-L67 | 🟢 良好 |
| 21 | HTTP handler 内直接 SD 阻塞 I/O（下载/写 config） | `web_server.c` L465、`config_manager.c` L250-L260 | 🟡 慢 SD 拖慢 Web |
| 22 | `recorder_start/stop/is_running` 幂等 + 持锁 | `data_recorder.c` L514-L576 | 🟢 良好 |
| 23 | `data_router_forward` 在 `uart_rx_task` 内同步 `user_uart_write_channel`（阻塞写），目标 TX 满会拖慢源收数 | `data_router.c` L56、`app_main.c` L65 | 🟡 |

## 五、日志与可观测性

| # | 问题 | 证据 | 级别 |
|---|---|---|---|
| 24 | 日志标签齐全（main/sd_card/usb_cdc/wifi_config/web_server/recorder/uart_driver/config/data_router） | 各 .c `#define TAG` | 🟢 良好 |
| 25 | 关键状态可经 `/api/status` 观测（drop/bytes/file_open/usb_owned 等） | `web_server.c` L115-L212 | 🟢 良好 |
| 26 | 丢字节无主动告警（仅计数） | `data_recorder.c` L301-L303 | 🟡 |

## 六、配置一致性（跨文件）

| # | 问题 | 证据 | 级别 |
|---|---|---|---|
| 27 | 文档 16MB vs sdkconfig 2MB | `docs/...报告.md` L322 vs `sdkconfig` L531 | 🔴 文档与配置矛盾 |
| 28 | 自定义 `partitions.csv` 未启用（SINGLE_APP 生效） | `partitions.csv` vs `sdkconfig` L545 | 🔴 |
| 29 | 控制台注释"UART1" vs sdkconfig UART0 vs CH1=UART0 | `console_cmd.c` L13 vs `sdkconfig` L1147 vs `uart_driver.h` L16 | 🔴 三处不一致 |
| 30 | 报告行数（app_main 183）vs 实际（201），内嵌代码为 v1.1 快照 | `build_tech_report.py` L18 vs `app_main.c` 实际 | 🟡 报告滞后 |

## 七、总体评价

- **优点**：模块边界清晰（12 个 .c 职责单一）、互斥/信号量使用规范（核心状态都有锁）、错误处理从 v1.1 的 abort 演进到 v1.2 的降级、API 有路径穿越防护、配置有回滚、可观测性好（status 全量 + 分级日志）。
- **最大短板**：① 配置一致性（Flash/分区/控制台三处矛盾）；② 事件循环阻塞（WiFi 重连 sleep）；③ 静默丢数据无告警。
- **面试建议**：主动承认 ① 是"归档时文档没同步配置"的工程疏漏，②③ 是"已知未修复项"，并给出修复方向（见 `09_OPTIMIZATION.md`）。

> 说明：本审计**只读不改**，未触碰任何源码。所有结论均可回源定位。
