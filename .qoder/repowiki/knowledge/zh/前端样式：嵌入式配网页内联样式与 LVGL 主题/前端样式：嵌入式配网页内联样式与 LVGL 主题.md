---
kind: frontend_style
name: 前端样式：嵌入式配网页内联样式与 LVGL 主题
category: frontend_style
scope:
    - '**'
source_files:
    - scripts/sonic_wifi_config.html
---

本仓库为 ESP32 语音助手固件工程，整体以 C/C++（ESP-IDF）为主，不包含传统 Web 前端项目中的 CSS/SCSS/Tailwind 等样式体系。与“前端风格”相关的视觉呈现集中在两个层面：

1. **配网工具页面**：`scripts/sonic_wifi_config.html` 是一个用于声波配网的单页 HTML，样式全部以内联 `<style>` 块编写，采用卡片式布局、圆角阴影、蓝色主按钮等基础 UI 风格，未引入任何外部 CSS 框架或设计系统。
2. **设备端显示层**：固件通过 `main/display/lvgl_display/` 下的 LVGL 驱动在板载屏幕渲染界面，UI 主题、字体、颜色等资源由 LVGL 工程配置及脚本 `scripts/Image_Converter/` 生成，属于嵌入式 GUI 主题而非 Web 样式。

因此，该仓库不存在跨模块统一的 Web 前端样式系统，也不存在集中管理的 CSS 主题文件。