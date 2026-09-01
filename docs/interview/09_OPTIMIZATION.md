# 09 · 优化清单（现状 → 改进 两栏）

> 只做分析、不改代码。每一条"现状"都可在代码定位；"改进"是可讨论的方向，未实际落地（除非标注）。

| # | 维度 | 现状（证据） | 改进方向 |
|---|---|---|---|
| 1 | 串口读模型 | 轮询式：RX task 每 100ms `uart_read_bytes`（`app_main.c` L60） | 改用 UART 事件队列（`uart_driver_install` 第 5 参传 queue，注册 `UART_DATA` 事件），降低空转与延迟 |
| 2 | 硬件流控 | 无流控 `UART_HW_FLOWCTRL_DISABLE`（`uart_driver.c` L57） | 高吞吐场景启用 RTS/CTS，配合 `uart_set_hw_flow_ctrl` |
| 3 | SD 写放大 | 16KB 缓冲 + ≥2KB 阈值，`allocation_unit_size=16KB`（`data_recorder.c` L21-L22、`sd_card.c` L64） | 用 `sd_test` 实测后调阈值/缓冲；对齐 FAT 簇减少写放大 |
| 4 | fsync 频率 | 每秒全量 `flush+fsync`（`data_recorder.c` L334-L366） | 按数据量自适应（有数据才 fsync），降低空闲期 SD 磨损 |
| 5 | live 拉取 | `recorder_live_fetch` 持锁线性扫 256 条 + 逐条 memcpy（`data_recorder.c` L620-L632） | 环形缓冲加单调 seq 索引，只拷贝 `since` 之后区间 |
| 6 | 实时传输 | 200ms 轮询（前端 `pollStream`） | 换 WebSocket（任务书 `/ws` 要求），或降频到 500ms + 合并批次 |
| 7 | 信号量语义 | `xSemaphoreGive(s_data_sem)` 每次写都给（`data_recorder.c` L305） | 用 task notification 或"有数据即置位"的 binary sem 严格语义 |
| 8 | 事件回调阻塞 | `wifi_event_handler` 内 `vTaskDelay(5000)`（`wifi_config.c` L51） | 回调只置 EventGroup，重连放独立任务/`esp_timer` |
| 9 | 配置存储安全 | WiFi 密码明文存 NVS + config.json（`config_manager.c` L152/L156） | NVS 加密分区（`CONFIG_NVS_ENCRYPTION`）或只存 STA 凭据不镜像到 SD |
| 10 | Web 安全 | 无鉴权、仅 HTTP（`WEB_PORT 80`，`web_server.c` L26） | AP 页加密码/一次性 token；局域网内才开放写接口 |
| 11 | 文件操作阻塞 | httpd 内 `fopen/fread` 下载、`config_manager_save` 写文件（`web_server.c` L465） | 下载放独立任务 + 流式缓冲，避免阻塞所有 Web 请求 |
| 12 | UART/控制台冲突 | CH1=UART0 与控制台 UART0 冲突（`uart_driver.h` L16 vs `sdkconfig` L1147） | 控制台改 UART1（或 CH1 改 UART1），对齐注释与 sdkconfig |
| 13 | 分区/Flash 矛盾 | 文档 16MB vs sdkconfig 2MB + SINGLE_APP（`sdkconfig` L531/L545） | 统一为量产 16MB OTA 布局并启用 `partitions.csv`，或明确文档标注 |
| 14 | 编译优化 | `CONFIG_COMPILER_OPTIMIZATION_DEBUG=y`（`sdkconfig` L559） | 量产切 `OPTIMIZATION_SIZE/PERF`，减小固件、提速 |
| 15 | 数据压缩 | raw `.bin` 无压缩（`data_recorder.h` L14） | 长记录可选 delta/帧压缩，或按时间分卷（rotating file） |
| 16 | 文件管理并发 | `max_files=5`（`sd_card.c` L63） | 提高 FATFS 打开文件数，或 Web 下载与记录分离 |
| 17 | 双通道波特率 | 两路共用同一 baud（`user_uart_init` 遍历同参） | 支持 CH1/CH2 独立波特率（`config` 拆 per-channel） |
| 18 | 错误可见性 | 丢字节只进 `drop_count`，无主动告警 | drop 超阈值时上报 status 字段/日志告警 |
| 19 | 断电恢复 | 记录文件无元数据，异常断电后无恢复标记 | 加文件头/尾校验（如写入块序号 + CRC），启动时校验 |
| 20 | 时间戳准确性 | 无 SNTP 同步，依赖系统时钟（`data_recorder.c` L114-L124） | 接入 SNTP（v1.3 "时间同步"目标），记录前同步 RTC |
| 21 | 单元测试 | 仓库无自动化测试（`.gitignore` 排除 `__pycache__` 等，未见 test） | 为 `ringbuf_*`、`config_manager` clamp/回滚 加 host 侧单测 |
| 22 | 组件化 | 12 个 .c 全在 `main/`（`main/CMakeLists.txt` L1） | 按任务书拆 `components/{wifi_manager,web_server,uart_bridge,recorder,...}` |

## 优先级建议（面试表述）

- **P0 正确性**：#8（事件回调阻塞）、#12（UART 冲突）、#9（密码明文）、#19（断电恢复）。
- **P1 性能**：#1/#2/#3/#4/#5/#7（采集-落盘链路）。
- **P2 工程化**：#13/#14/#21/#22（分区一致性、编译选项、测试、组件化）。

> 面试陷阱提醒：优化题要"先测后优"——本仓库有 `sd_speed_test`（`esp> sd_test`）可测 SD 吞吐，但**没有任何实测吞吐/丢包数据入库**，所以回答时讲"我会先用 sd_test + /api/status 的 drop_count 定位瓶颈，再决定改哪一层"，而不是直接给结论。
