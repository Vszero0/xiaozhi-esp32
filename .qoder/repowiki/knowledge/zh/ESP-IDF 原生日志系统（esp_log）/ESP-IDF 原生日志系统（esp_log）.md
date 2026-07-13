---
kind: logging_system
name: ESP-IDF 原生日志系统（esp_log）
category: logging_system
scope:
    - '**'
source_files:
    - main/main.cc
    - main/application.cc
    - sdkconfig.defaults
    - main/boards/common/esp_video.cc
    - main/display/lvgl_display/jpg/jpeg_to_image.c
    - main/boards/esp-hi/config.json
    - main/boards/waveshare/esp32-s3-touch-lcd-4.3c/sdkconfig.4_3c
---

本仓库采用 ESP-IDF 内置的 esp_log.h 作为统一日志框架，未引入第三方日志库。所有模块通过 ESP_LOG* 宏输出结构化文本日志，并通过 Kconfig + sdkconfig.defaults 集中控制默认级别与控制台通道。

1. 使用的系统与工具
- 框架：ESP-IDF esp_log.h，提供 ESP_LOGE/W/I/D/V 五级日志宏。
- 控制台输出：通过 CONFIG_ESP_CONSOLE_UART_DEFAULT 或 CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG 选择 UART0 或 USB Serial/JTAG；部分板型在 boards/*/config.json 中强制启用 USB JTAG 以便调试。
- 构建期开关：LOG_LOCAL_LEVEL 可在文件级提升最低输出级别，配合 CONFIG_LOG_DEFAULT_LEVEL 使用。

2. 关键文件与位置
- 入口初始化：main/main.cc、main/application.cc（包含大量业务日志）。
- 全局默认配置：sdkconfig.defaults（关闭 Bootloader 日志、设置分区表等）。
- 按板型覆盖控制台/默认级别：main/boards/<board>/config.json、main/boards/<board>/sdkconfig.*。
- 文件级 DEBUG 开关示例：main/boards/common/esp_video.cc、main/display/lvgl_display/jpg/jpeg_to_image.c。

3. 架构与约定
- TAG 命名：每个源文件定义 static const char* TAG = "<模块名>";，如 "main"、"EspVideo"、"PowerManager"、"GpioManager" 等，便于过滤。
- 级别策略：
  - 默认全局级别由 CONFIG_LOG_DEFAULT_LEVEL 决定（常见为 INFO），Bootloader 日志关闭以节省空间。
  - 需要更详细输出的模块在编译前 #define LOG_LOCAL_LEVEL MAX(CONFIG_LOG_DEFAULT_LEVEL, ESP_LOG_DEBUG)，并在运行时调用 esp_log_level_set(TAG, ESP_LOG_DEBUG) 动态开启。
- 无自定义封装：代码直接使用 ESP_LOGI/E/W/D 宏，未对日志进行二次包装或结构化序列化。

4. 开发者应遵循的规则
- 在每个 .cc/.h 顶部定义 static const char* TAG = "<模块名>";，并 #include <esp_log.h>。
- 优先使用 ESP_LOGI 记录正常流程，ESP_LOGW 记录可恢复异常，ESP_LOGE 仅用于错误路径；避免在热路径使用 ESP_LOGD。
- 如需临时打开某模块的详细日志，采用“编译期 LOG_LOCAL_LEVEL + 运行期 esp_log_level_set(TAG, ...)”组合，不要直接修改全局 CONFIG_LOG_DEFAULT_LEVEL。
- 新增板型时，若需串口调试，应在 boards/<board>/config.json 中添加 CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y，并确保 CONFIG_LOG_DEFAULT_LEVEL 至少为 INFO。
- 不要在高频循环内打印大对象字符串，必要时先判断日志级别再构造消息。