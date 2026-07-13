---
kind: configuration_system
name: ESP-IDF Kconfig + NVS 分层配置体系
category: configuration_system
scope:
    - '**'
source_files:
    - main/Kconfig.projbuild
    - sdkconfig.defaults
    - main/CMakeLists.txt
    - main/settings.h
    - main/settings.cc
    - main/boards/bread-compact-wifi/config.json
    - main/boards/waveshare/esp32-p4-nano/config.json
    - scripts/release.py
    - partitions/v2/16m.csv
---

本仓库采用 ESP-IDF 原生 Kconfig 作为编译期配置主入口，结合运行时 NVS 持久化与板级 `config.json` 构建描述文件，形成“编译期宏 + 运行时键值”的分层配置体系。

## 1. 系统/框架
- **Kconfig.projbuild**：在 `menu “Xiaozhi Assistant”` 下集中定义所有应用级选项（网络类型、板型、语言、唤醒词、音频处理、显示风格、WiFi 配网方式等），通过 `depends on` 与目标芯片绑定，实现条件编译开关。
- **sdkconfig.defaults / sdkconfig.defaults.*_target**：提供全局默认值（分区表 v2、LVGL 裁剪、WDT、mbedtls 优化等），各 target 可覆盖。
- **CMakeLists.txt (main)**：根据 `CONFIG_BOARD_TYPE_*` 选择具体板目录、字体、emoji 集，并动态生成语言头 `assets/lang_config.h`。
- **scripts/release.py**：扫描 `main/boards/**/config.json`，解析 `manufacturer/target/builds[].sdkconfig_append`，驱动 `idf.py set-target` 和增量编译，是跨板批量构建的编排器。
- **NVS Settings 类**：`main/settings.{h,cc}` 封装 NVS namespace 读写，用于设备 UUID、用户偏好等运行时持久化。

## 2. 关键文件与包
- `main/Kconfig.projbuild` — 全部菜单项与依赖关系
- `sdkconfig.defaults`、`sdkconfig.defaults.esp32*` — 全局与目标默认配置
- `main/CMakeLists.txt` — 按 BOARD_TYPE 映射到源码/资源
- `main/settings.h` / `main/settings.cc` — NVS 键值存取 API
- `main/boards/<board>/config.json` — 单板构建变体清单（target + sdkconfig_append）
- `scripts/release.py` — 基于 config.json 的自动化构建脚本
- `partitions/v2/*.csv` — 运行时分区布局（OTA、assets 等）

## 3. 架构与约定
- **编译期配置**：通过 menuconfig 或 `idf.py set <option>=y` 设置 Kconfig 符号；每个板型在 `config.json` 中以 `sdkconfig_append` 声明差异，避免手写 sdkconfig。
- **运行时配置**：以 `Settings("namespace")` 打开 NVS，使用 `GetString/SetString`、`GetInt/SetInt`、`GetBool/SetBool` 存取；析构时自动 commit，支持只读模式回退默认值。
- **板型抽象**：`Board` 基类 + `DECLARE_BOARD(BOARD_CLASS_NAME)` 宏，由 CMake 选中的 `<board>.cc` 提供实例；`Board::GetSystemInfoJson()` 汇总分区、OTA、显示等信息供协议上报。
- **语言与资源**：Kconfig `LANGUAGE_*` 决定 `LANG_DIR`，CMake 收集对应 `assets/locales/<dir>/*.ogg` 并生成 `lang_config.h`，非 en-US 时自动 fallback 到 en-US 缺失文件。
- **分区表**：默认启用 v2 布局（`partitions/v2/16m.csv`），为 assets 与双 OTA 预留空间，不同 Flash 尺寸在 `partitions/v2/` 下提供多套 CSV。

## 4. 开发者规则
- 新增板型：在 `main/boards/<vendor>/<board>/` 下创建 `config.h`、`<board>_board.cc`、`config.json`，并在 `Kconfig.projbuild` 的 `choice BOARD_TYPE` 中添加条目（带 `depends on IDF_TARGET_xxx`）。
- 不要直接修改原板目录；如需 IO 差异，新建板目录或在 `config.json` 的 `builds[]` 中用不同 `name` + `sdkconfig_append` 产出独立固件名。
- 运行时用户数据一律通过 `Settings` 类写入指定 namespace，禁止裸写 NVS；需要新字段时在 `settings.h` 暴露对应 Get/Set 方法。
- 构建变体优先写在 `config.json` 的 `sdkconfig_append`，而非手动维护 sdkconfig 文件；release.py 会校验 `manufacturer` 与目录层级一致性。
- 语言切换仅通过 Kconfig `LANGUAGE_*`，不要在运行时硬编码 locale 路径。