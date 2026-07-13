---
kind: dependency_management
name: ESP-IDF 组件依赖管理（idf_component.yml + CMake）
category: dependency_management
scope:
    - '**'
source_files:
    - main/idf_component.yml
    - CMakeLists.txt
    - main/CMakeLists.txt
    - scripts/Image_Converter/requirements.txt
    - scripts/p3_tools/requirements.txt
---

本仓库基于 ESP-IDF 构建系统，采用 ESP-IDF Component Manager 作为统一的第三方依赖声明与解析机制，辅以 CMake/Kconfig 按目标芯片和板型动态裁剪源码。

### 1. 使用的系统与工具
- ESP-IDF Component Manager：通过 main/idf_component.yml 集中声明所有外部组件及其版本约束；构建时由 IDF 自动下载、缓存并链接。
- CMake + Kconfig：根级 CMakeLists.txt 引入 project.cmake 并启用 MINIMAL_BUILD，main/CMakeLists.txt 根据 CONFIG_BOARD_TYPE_*、CONFIG_IDF_TARGET_* 等 Kconfig 宏动态选择板级源文件与组件。
- Python 脚本辅助：scripts/ 下各子目录的 requirements.txt 用于本地开发工具链（图像转换、P3 音频处理等），与固件运行时依赖解耦。

### 2. 关键文件与位置
- main/idf_component.yml：项目唯一的外部组件清单，包含 LVGL、esp-sr、esp_lcd_*、esp_audio_codec、esp_wifi_connect、xiaozhi-fonts 等 40+ 个组件，并按 target 使用 rules.if 做条件依赖。
- CMakeLists.txt（根）：设置 PROJECT_VER、idf_build_set_property(MINIMAL_BUILD ON)，统一入口。
- main/CMakeLists.txt：通过 idf_build_get_property(BUILD_COMPONENTS) 与自定义 find_component_by_pattern 函数在构建期探测已拉取的组件路径，再按 board 类型追加对应 .cc/.c 源文件。
- sdkconfig.defaults*：为不同 ESP32 系列提供默认 Kconfig 值，间接决定哪些组件被编译进最终固件。
- scripts/*/requirements.txt：仅影响本地 Python 工具，不参与固件构建。

### 3. 架构与约定
- 单一声明点：所有运行时依赖集中在 main/idf_component.yml，禁止在 CMake 中直接写死 URL 或版本号。
- 条件依赖：大量使用 version: ^x.y.z + rules.if: target in [...] 实现同一份代码适配多芯片，如 esp32-camera 仅在 esp32s3 上启用。
- 精确锁定：对破坏性升级敏感的组件（如 esp_video ==1.3.1、esp_lcd_ili9341 ==1.2.0）使用 == 固定版本，并在注释说明升级需同步改代码。
- 最小化构建：MINIMAL_BUILD ON 确保只编译 main 及它显式依赖的组件，避免把整个 IDF 树打进去。
- 板级隔离：新增硬件只需在 boards/<vendor>/<board>/ 下添加配置，并通过 main/CMakeLists.txt 中的 CONFIG_BOARD_TYPE_* 分支接入，不改动通用依赖。

### 4. 开发者应遵循的规则
1. 新增依赖必须写入 main/idf_component.yml，写明 version 约束，必要时加 rules.if 限定目标芯片。
2. 敏感组件用 == 锁定，并在注释中记录升级需同步修改的代码位置。
3. 不要手动修改 build/ 下的组件缓存，依赖应由 Component Manager 自动拉取；若网络受限，参考 README 切换镜像源。
4. Kconfig 控制开关：通过 sdkconfig.defaults.* 或 menuconfig 开启/关闭功能，让 CMake 自动增减源文件，而不是硬编码 if(CONFIG_XXX) 到源码里。
5. Python 工具依赖独立维护：scripts/*/requirements.txt 与固件无关，更新时仅影响本地环境，不影响 CI 构建。