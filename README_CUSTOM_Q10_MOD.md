# Keychron Q10 (ANSI Encoder) Vial-QMK 编译与定制开发指南

本文档记录了基于 `vial-qmk` 源码为 Keychron Q10（旋钮版 / ANSI 配列）从零搭建编译环境，并进行个性化定制的全过程。

## 🛠 第一阶段：从零构建环境与编译排坑 (macOS 篇)

在全新 macOS 环境下，由于工具链依赖和安全策略限制，首次编译需要跨越几个经典障碍。

### 1. 基础环境准备

为了保持全局 Python 环境纯净，我们使用 `uv` 来隔离安装 QMK 命令行工具：

```bash
uv tool install qmk

```

拉取目标仓库源码并进入目录：

```bash
git clone https://github.com/<你的用户名>/vial-qmk.git
cd vial-qmk

```

### 2. 绕过 Homebrew 安全拦截 (核心避坑点)

新版 Homebrew 会拦截未经手动信任的第三方 Tap 源，导致 `arm-none-eabi-gcc` 等交叉编译器下载失败。在执行 `qmk setup` 之前，**必须先打通权限**：

```bash
# 添加并信任所有必需的硬件编译源
brew tap osx-cross/arm
brew tap osx-cross/avr

brew trust qmk/qmk
brew trust osx-cross/arm
brew trust osx-cross/avr

```

### 3. 初始化 QMK 工作区与同步子模块

执行环境安装命令：

```bash
qmk setup -H .

```

> **注意：** 遇到 `What do you want to do?` 提示时，**必须选 `3` (Keep it and continue)**，以防代码被官方原版覆盖。

**暴力修复 Git 子模块（解决 LUFA 驱动报错）：**
若遇到 `HIDClassCommon.h` 找不到的问题，说明底层依赖下载缺失，需强行洗刷并拉取：

```bash
git submodule deinit -f --all
git submodule update --init --recursive --force
rm -rf .build
qmk clean

```

最后运行 `qmk doctor`，只要 `arm-none-eabi-gcc` 等工具显示绿色 `Ψ` 即代表环境彻底就绪。

### 4. 首次编译命令

原作者默认的无 Vial 映射名为 `keychron`，**必须指定 `-km vial` 才能开启完整的 Vial 界面支持与配置文件映射**：

```bash
qmk compile -kb keychron/q10/ansi_encoder -km vial

```

---

## 🚀 第二阶段：固件深度定制与源码修改全纪录

为了解决官方 Launcher 误判、层数受限以及 UI 错位等问题，对源码进行了以下 4 处核心修改（已在你 Fork 的仓库中生效）：

### 1. 解除层数限制 (扩展至 8 层)

* **文件路径**：`keyboards/keychron/q10/config.h`
* **修改内容**：在文件末尾追加定义，解锁 8 层空间，完美兼容包含 Layer 4（数字小键盘层）的旧配置文件。

```c
/* --- 将动态键映射层数设置为 8 --- */
#define DYNAMIC_KEYMAP_LAYER_COUNT 8

```

### 2. 伪装版本号 (欺骗网页端驱动)

* **文件路径**：`keyboards/keychron/q10/info.json`
* **修改内容**：在 `"usb"` 块中将版本号写死为 `1.0.4`。此单处修改即可防止 Keychron Launcher 网页端将其误识别为官方 2.0 固件。

```json
    "usb": {
        "vid": "0x3434",
        "device_version": "1.0.4"
    },

```

### 3. 自定义冷白偏蓝色切层指示灯

* **文件路径**：`keyboards/keychron/q10/ansi_encoder/keymaps/vial/keymap.c`
* **修改内容**：接管底层矩阵灯光函数。当切换至 Layer 4 时，旋钮下方第一个中转按键（矩阵坐标 `[1][0]`）将自动亮起通透的高亮冰白色；切回其他层时自动恢复正常。

```c
bool rgb_matrix_indicators_user(void) {
    if (IS_LAYER_ON(4)) {
        uint8_t led_index = g_led_config.matrix_co[1][0];

        if (led_index != NO_LED) {
            // R=150, G=210, B=255：物理灯珠呈现冷白偏蓝色（冰白）
            rgb_matrix_set_color(led_index, 150, 210, 255);
        }
    }
    return false;
}

```

### 4. 完美修复 Vial UI 旋钮坐标

* **文件路径**：`keyboards/keychron/q10/ansi_encoder/keymaps/vial/vial.json`
* **修改内容**：将 `"0,0\n\n\n\n\n\n\n\n\ne"` 与 `"0,1\n\n\n\n\n\n\n\n\ne"` 两个旋转动作图标抽离并置于 `keymap` 阵列的第一行最前。在其后通过 `{"x": 0.25}` 找补间距，确保右侧的物理 `Mute` 静音键能完美锚定在原始 `2.25` 坐标上，解决图标乱飘问题。

---

## 🧹 第三阶段：日常维护与排错连招 (终极闭环)

当你修改了上述代码（特别是 JSON 文件）再次编译，或者在使用 Vial 切换层时遇到诸如 `KeyError` 等内存崩坏弹窗时，**必须严格执行以下三步曲**：

### 1. 清除幽灵缓存并纯净编译

```bash
qmk clean
qmk compile -kb keychron/q10/ansi_encoder -km vial

```

### 2. 物理强制抹除硬件 EEPROM 并烧录

这一步是解决内存区块错位（导致 Vial 报错）的核心：

1. 拔掉键盘 USB 数据线。
2. **按住键盘物理最左上角的第一个按键（即 `Mute` 静音键）不松手**。
3. 插上数据线，保持按住 3～5 秒，直到键盘全灭进入 DFU 模式。
4. 在终端运行烧录命令（或使用 QMK Toolbox 刷入新生成的 `.bin`）：

```bash
qmk flash -kb keychron/q10/ansi_encoder -km vial

```

### 3. 生成终极无损配置文件

固件层数扩展和 UI 重对齐后，旧的 Vial 备份文件可能会产生兼容问题。
刷入纯净固件并彻底重置后，关闭并重新打开 Vial 软件（确认全绿无错），立即点击 **File -> Save current layout**。
以后若键盘复位或清空，只需直接 Load 这份新生成的 `.json` 文件即可一键完美复原。
