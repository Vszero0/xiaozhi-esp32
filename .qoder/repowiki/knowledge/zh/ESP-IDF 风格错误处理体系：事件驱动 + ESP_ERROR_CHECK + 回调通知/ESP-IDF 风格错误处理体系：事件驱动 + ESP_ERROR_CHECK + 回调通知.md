---
kind: error_handling
name: ESP-IDF 风格错误处理体系：事件驱动 + ESP_ERROR_CHECK + 回调通知
category: error_handling
scope:
    - '**'
source_files:
    - main/application.h
    - main/application.cc
    - main/protocols/protocol.h
    - main/protocols/protocol.cc
    - main/protocols/mqtt_protocol.cc
    - main/protocols/websocket_protocol.cc
    - main/assets.cc
    - main/audio/codecs/box_audio_codec.cc
---

本仓库基于 ESP-IDF 构建，采用“底层断言式检查 + 上层事件/回调传播”的分层错误处理模式，贯穿协议、OTA、音频与板级适配各层。

## 1. 系统/框架层面

- **ESP-IDF 宏 `ESP_ERROR_CHECK`**：在大量初始化路径（I2S、ADC、esp_timer、esp_codec_dev 等）中直接调用，失败即 panic，适用于“启动期不可恢复”的错误。
- **`esp_err_t` 返回值**：网络、分区表读写、OTA 升级等可预期失败的路径统一返回 `esp_err_t`，由调用方判断并做重试或降级。
- **FreeRTOS EventGroup + 主循环**：应用核心通过 `Application::Run()` 的 `xEventGroupWaitBits` 轮询事件位，包括 `MAIN_EVENT_ERROR`，实现跨任务错误上报。
- **Protocol 抽象层的 `OnNetworkError` / `SetError`**：所有协议实现（MQTT、WebSocket）将网络侧错误封装为字符串消息，通过回调上抛到 Application，再由主循环统一展示。

## 2. 关键文件与位置

- `main/application.h` / `main/application.cc`：定义 `MAIN_EVENT_ERROR`、`last_error_message_`，并在主循环中消费错误事件，调用 `Alert` 向用户反馈。
- `main/protocols/protocol.h` / `main/protocols/protocol.cc`：定义 `OnNetworkError` 回调注册接口与 `SetError` 默认实现，是错误从协议层冒泡的统一出口。
- `main/protocols/mqtt_protocol.cc` / `main/protocols/websocket_protocol.cc`：具体协议在连接失败、发送失败、超时等场景调用 `SetError(Lang::Strings::...)`，触发上层错误流程。
- `main/assets.cc`：Flash assets 分区 mmap/erase/write 使用 `esp_err_t` + `ESP_LOGE` 记录失败详情。
- `main/audio/codecs/box_audio_codec.cc` 及各 board 的 power_manager.h：大量使用 `ESP_ERROR_CHECK` 包裹硬件初始化，确保启动期失败快速暴露。

## 3. 架构与约定

- **分层职责**：
  - 底层驱动/外设：用 `ESP_ERROR_CHECK` 表达“绝对不应失败”的假设；
  - 中间层（assets、audio codec）：返回 `esp_err_t` 并配合 `ESP_LOGE/W/I` 输出上下文；
  - 协议层：将网络异常归一化为 `Lang::Strings::*` 本地化文本，通过 `SetError` → `OnNetworkError` 回调上抛；
  - 应用层：保存 `last_error_message_`，置位 `MAIN_EVENT_ERROR`，在主循环中统一 `Alert` 显示。
- **错误呈现**：通过 `Application::Alert(status, message, emotion, sound)` 同时更新屏幕状态栏、聊天消息与提示音，保证用户在无 UI 环境下也能感知错误。
- **可恢复性**：OTA 版本检查、激活流程内部自行处理 `esp_err_t` 并带指数退避重试；网络断开后自动关闭音频通道并回退空闲态。

## 4. 开发者应遵循的规则

1. **硬件初始化**：一律使用 `ESP_ERROR_CHECK`，不要吞掉返回值。
2. **可预期失败**：返回 `esp_err_t`，并在日志中使用 `esp_err_to_name(err)` 打印可读名称。
3. **网络/协议错误**：通过 `SetError(Lang::Strings::XXX)` 上报，不要直接 `printf` 或抛 C++ 异常。
4. **跨任务错误**：如需从 ISR/回调上报，设置 `MAIN_EVENT_ERROR` 并写入 `last_error_message_`，交由主循环处理。
5. **用户可见错误**：必须走 `Alert`，附带情绪图标与提示音，保持多模态一致体验。
6. **避免裸 `panic`**：仅在“设备不可能继续运行”的极端路径使用 `ESP_ERROR_CHECK`，其余情况尽量可恢复。
