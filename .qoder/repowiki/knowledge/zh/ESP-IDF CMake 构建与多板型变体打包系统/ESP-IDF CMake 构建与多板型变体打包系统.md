---
kind: build_system
name: ESP-IDF CMake 构建与多板型变体打包系统
category: build_system
scope:
    - '**'
source_files:
    - CMakeLists.txt
    - main/CMakeLists.txt
    - main/idf_component.yml
    - main/Kconfig.projbuild
    - sdkconfig.defaults
    - sdkconfig.defaults.esp32
    - sdkconfig.defaults.esp32c3
    - sdkconfig.defaults.esp32s3
    - sdkconfig.defaults.esp32p4
    - partitions/v2/16m.csv
    - .github/workflows/build.yml
    - scripts/release.py
---

## 1. 构建系统与工具链

- 基础框架：基于 ESP-IDF v5.5+，使用 CMake + Kconfig 作为核心构建系统。根 CMakeLists.txt 仅引入 IDF 的 project.cmake、启用 MINIMAL_BUILD 并声明 PROJECT_VER。
- 组件管理：通过 main/idf_component.yml 集中声明所有第三方组件（LVGL、esp_lcd_*、esp_sr、esp32-camera、esp_wifi_connect 等），并按目标芯片用 rules.if target in [...] 做条件依赖。
- 分区表：partitions/v1/ 与 partitions/v2/ 两套 CSV 布局，分别对应带 model 分区的旧版与引入 assets 分区、双 OTA 的新版；默认指向 v2/16m.csv。
- Kconfig 配置：sdkconfig.defaults 提供全局优化与 LVGL 裁剪；各芯片族在 sdkconfig.defaults.esp32* 中覆盖默认值。

## 2. 多板型变体机制

- 板型目录结构：main/boards/<manufacturer>/<board>/ 下每个板子包含 config.h、config.json、*.cc 以及可选的 power_manager.h 等。common/ 存放跨板公共实现。
- Kconfig 选择：main/Kconfig.projbuild 为每个板型定义 CONFIG_BOARD_TYPE_xxx=y 符号；main/CMakeLists.txt 中巨大的 if(CONFIG_BOARD_TYPE_*) 分支将 Kconfig 映射到 BOARD_TYPE、MANUFACTURER、字体与 emoji 集合等编译期常量。
- 资源与语言：根据 CONFIG_LANGUAGE_* 选择 assets/locales/<lang>/ 下的 OGG 音频与 language.json，缺失项回退 en-US。

## 3. 自动化构建脚本与 CI

- 统一入口 scripts/release.py：
  - 扫描 main/boards/**/config.json，解析 target、builds[].name、sdkconfig_append，自动生成变体矩阵。
  - 自动推导 CONFIG_BOARD_TYPE_xxx 符号（支持显式指定、按 target 匹配、按名称模糊匹配）。
  - 调用 idf.py set-target <target> → idf.py -DBOARD_NAME=... -DBOARD_TYPE=<board> build → idf.py merge-bin → 输出 releases/v{version}_{name}.zip。
  - 内置 _AUTO_SELECT_RULES 模拟 Kconfig select 行为（如 BLUFI 自动开启 BT 相关选项）。
- CI 流水线 .github/workflows/build.yml：
  - push main 时全量构建；PR 场景基于 diff 文件路径计算受影响 board，再并行矩阵构建。
  - 容器镜像 espressif/idf:v5.5.2，产物上传为 build/merged-binary.bin。

## 4. 辅助构建工具

- scripts/spiffs_assets/：SPIFFS/LVGL 资源打包生成器。
- scripts/Image_Converter/、scripts/ogg_converter/、scripts/p3_tools/：图像、音频格式转换与 P3 格式处理。
- scripts/gen_lang.py、scripts/build_default_assets.py：语言头与默认资源预构建。

## 5. 开发者约定

- 新增板型需在 main/boards/<manufacturer>/<board>/ 下提供 config.json（含 target、builds[]）与 config.h，并在 main/CMakeLists.txt 中添加对应 CONFIG_BOARD_TYPE_xxx 分支。
- 若同一 leaf name 存在多个 Kconfig 符号，应在 config.json 的 sdkconfig_append 中显式指定 CONFIG_BOARD_TYPE_xxx=y，避免歧义。
- 组件版本锁定在 idf_component.yml，新增依赖需遵循 rules.if target in [...] 条件化声明。
- 项目版本号集中在根 CMakeLists.txt 的 set(PROJECT_VER ...)，由 release.py 自动读取用于 zip 命名。