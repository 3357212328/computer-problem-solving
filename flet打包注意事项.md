# Flet + Nuitka 桌面程序打包注意事项与实战指南

本文档总结了在 Windows 环境下使用 **Nuitka**（配合 `uv` 虚拟环境与 `UPX`）将 **Flet (0.85+ / 1.x)** 桌面应用程序（含 `pyserial` 串口与 `bleak` BLE 蓝牙）打包为**单文件、无黑框、小体积 `.exe`** 的完整配置与排坑经验。

---

## 一、 常见报错与核心原因速查表

| 序号 | 打包后运行时的典型现象 / 报错信息 | 根本原因 | 解决方案 |
| :--- | :--- | :--- | :--- |
| 1 | `ModuleNotFoundError: No module named 'msgpack'`（或缺少 `flet_desktop`、`httpx`、`oauthlib`、`repath`） | Nuitka 静态分析未递归收录 `flet` 底层依赖，或打包时误选了系统全局 Python 而非项目的 `.venv` 解释器，或误将 `flet` 目录当普通数据文件夹拷贝 | 1. 使用 `uv run python -m nuitka` 确保使用 `.venv` 环境<br>2. 添加 `--include-package=flet,flet_desktop,msgpack,httpx,oauthlib,repath`<br>3. 在 `main.py` 顶部显式 `import` 这些底层包 |
| 2 | 窗口白屏红字报错：`[Errno 2] No such file or directory: '.../flet/controls/material/icons.json'` | Flet 0.85+ 在运行时会动态读取包内的 `icons.json` 与 `cupertino_icons.json`，而 `--include-package=flet` 默认只编译 `.py` 代码，不拷贝 `.json` 数据文件 | 必须加上 **`--include-package-data=flet`**，将 Flet 包内的 JSON 图标表一并打入包中 |
| 3 | 蓝牙扫描或连接无反应 / 报 `winrt` 后端找不到 | `bleak` 在 Windows 下通过动态导入加载 `bleak.backends.winrt` 及 `winrt.*` 原生扩展 | 添加 `--include-package=bleak` 与 `--include-package=winrt`，并在代码中显式导入 `bleak.backends.winrt.client` 和 `scanner` |
| 4 | 打包生成的 `.exe` 体积高达 **100 MB+** | 1. 虚拟环境未安装 `zstandard`，导致 `--onefile` 退化为 **0% 压缩率** 的原始拼接<br>2. 默认打包了整套 Windows C++ Runtime DLL<br>3. 未启用 UPX | 1. 执行 `uv add --dev zstandard` 开启 Zstd 压缩（体积立减 75%，降至 **~25 MB**）<br>2. 添加 `--include-windows-runtime-dlls=no`<br>3. 启用 `--enable-plugin=upx` |
| 5 | `FATAL: multiprocessing: Conflict between user and plugin decision for module 'multiprocessing'` | 在 `--nofollow-import-to` 中手动排除了 `multiprocessing`，与 Nuitka 内置插件冲突 | 不要在 `--nofollow-import-to` 中排除 `multiprocessing` |
| 6 | 窗口左上角标题栏或 Windows 任务栏依然显示默认的粉色 Flet 图标 | Flet 桌面端底层启动的是 `~/.flet/client/.../flet.exe` 宿主进程，其内置资源图标为粉色 Flet 图标；仅靠 `page.window.icon` 在部分 Windows 10/11 环境下无法覆盖任务栏类图标 | 1. 生成包含 `16/24/32/48/64`（32-bit BGRA DIB）+ `256`（PNG）的标准多尺寸 `.ico`<br>2. 设置 `page.window.icon`<br>3. 使用 `ctypes.windll.user32` 发送 `WM_SETICON` 并调用 `SetClassLongPtrW(GCLP_HICON)` 强制刷新窗口与任务栏图标 |
| 7 | `--onefile` 模式下配置文件修改后下次启动丢失，或找不到 `assets` 图标 | `--onefile` 启动时会解压到 `C:\Users\...\AppData\Local\Temp\onefile_xxx\` 临时目录，关闭后该目录被清理 | **必须区分两个目录**：<br>• **读写配置目录 (`APP_DIR`)**：取 `.exe` 所在真实目录（`Path(sys.argv[0]).resolve().parent`）<br>• **内置静态资源目录 (`BUNDLE_DIR`)**：取解压临时目录（`Path(__file__).resolve().parent`） |

---

## 二、 环境准备清单

在项目根目录下确保已安装 `nuitka` 和 **`zstandard`**（决定单文件体积能否从 100MB 降到 25MB 的关键）：

```powershell
# 1. 确保安装 zstandard（用于 Nuitka --onefile 高倍率压缩）
uv add --dev zstandard

# 2. 检查当前虚拟环境中的 Nuitka 与 C 编译器 (MSVC cl.exe 或 MinGW64)
uv run python -m nuitka --version
```

> **提示**：首次在机器上运行 Nuitka `--onefile` 时，Nuitka 可能会自动下载缓存工具 `depends.exe`（Dependency Walker），建议带上 `--assume-yes-for-downloads` 参数避免交互卡住。

---

## 三、 推荐的 `main.py` 源码级配置模板

Nuitka 支持直接在入口文件 `main.py` 最顶部编写 `# nuitka-project:` 注释指令。这样配置后，无论通过命令行还是脚本调用，都不需要每次手敲一长串参数。

### 1. 顶部 Nuitka 编译指令（放在 `main.py` 第 1 行）

```python
# nuitka-project: --onefile
# nuitka-project: --windows-console-mode=disable
# nuitka-project: --follow-imports
# nuitka-project: --include-windows-runtime-dlls=no
# nuitka-project: --enable-plugin=upx
# nuitka-project: --upx-binary=D:/Python310/upx-5.1.0-win64/upx.exe
# nuitka-project: --include-package=flet
# nuitka-project: --include-package-data=flet
# nuitka-project: --include-package=flet_desktop
# nuitka-project: --include-package=msgpack
# nuitka-project: --include-package=httpx
# nuitka-project: --include-package=oauthlib
# nuitka-project: --include-package=repath
# nuitka-project: --include-package=bleak
# nuitka-project: --include-package=winrt
# nuitka-project: --include-package=serial
# nuitka-project: --nofollow-import-to=unittest,pydoc,doctest,tkinter,sqlite3,xmlrpc,curses,lib2to3,ensurepip,venv,turtledemo,turtle,idlelib
# nuitka-project: --include-data-dir={MAIN_DIRECTORY}/assets=assets
# nuitka-project: --windows-icon-from-ico={MAIN_DIRECTORY}/assets/icon.ico
# nuitka-project: --output-dir={MAIN_DIRECTORY}/output
# nuitka-project: --output-filename=机械臂全功能控制套件.exe
# nuitka-project: --remove-output
```

### 2. 显式导入隐藏依赖（防止静态分析漏包）

```python
import bleak
import flet as ft
import flet_desktop  # noqa: F401 - 确保收录桌面端运行时依赖
import httpx         # noqa: F401 - 确保收录 flet 底层依赖
import msgpack       # noqa: F401 - 确保收录 flet.messaging.protocol 依赖
import oauthlib      # noqa: F401 - 确保收录 flet.auth 依赖
import repath        # noqa: F401 - 确保收录 flet 路由依赖
import serial
import serial.tools.list_ports

try:
    import bleak.backends.winrt.client   # noqa: F401
    import bleak.backends.winrt.scanner  # noqa: F401
except Exception:
    pass
```

### 3. 正确区分「配置文件路径」与「打包内置资源路径」

```python
import os
import sys
from pathlib import Path


def get_app_dir() -> Path:
    """获取 .exe 所在真实目录（用于读写 JSON 配置文件、导出日志等持久化数据）。"""
    if getattr(sys, "frozen", False):
        return Path(sys.executable).resolve().parent
    if "__compiled__" in globals():
        # Nuitka --onefile 模式下 sys.argv[0] 指向用户双击的 .exe 真实路径
        return Path(sys.argv[0]).resolve().parent
    if os.environ.get("SERIOUS_PYTHON_SITE_PACKAGES"):
        return Path(sys.executable).resolve().parent
    return Path(__file__).resolve().parent


def get_bundle_dir() -> Path:
    """获取打包内置静态资源目录（Nuitka --onefile 下指向 Temp 解压目录）。"""
    if hasattr(sys, "_MEIPASS"):
        return Path(getattr(sys, "_MEIPASS")).resolve()
    return Path(__file__).resolve().parent


APP_DIR: Path = get_app_dir()
BUNDLE_DIR: Path = get_bundle_dir()

# 图标等只读资源优先从包内解压目录读取；配置文件固定保存在 .exe 同级目录
ASSETS_DIR: Path = (BUNDLE_DIR / "assets") if (BUNDLE_DIR / "assets").exists() else (APP_DIR / "assets")
SETTINGS_FILE: Path = APP_DIR / "arm_suite_config.json"
```

---

## 四、 一键打包命令

当 `main.py` 顶部已经包含上述 `# nuitka-project:` 指令后，日常打包只需在项目根目录下执行一行命令：

```powershell
uv run python -m nuitka --assume-yes-for-downloads main.py
```

- **首次编译耗时**：约 10 ~ 15 分钟（需要编译约 1400 个 C 文件，Nuitka 会自动通过 `clcache` 缓存编译结果）。
- **二次及后续编译耗时**：约 **2 ~ 5 分钟**（`clcache` 命中率通常在 99% 以上，仅重新编译修改过的 `main.py` 并重新打包单文件镜像）。
- **最终产物**：`D:\flet_prj\arm_contral\output\机械臂全功能控制套件.exe`（约 **25.7 MB**，单文件免安装，无命令行黑框）。

---

## 五、 关于跨电脑分发（离线电脑运行说明）

Flet 的桌面端架构分为两部分：
1. **Python 后端主程序**：已被 Nuitka 完整编译进 `机械臂全功能控制套件.exe`。
2. **Flutter 桌面渲染引擎 (`flet.exe` 客户端)**：
   - 默认存放在用户目录：`C:\Users\<用户名>\.flet\client\flet-desktop-full-0.85.1\flet\`。
   - **联网电脑**：首次双击运行 `.exe` 时，若目标电脑没有该目录，`flet_desktop` 会自动从 GitHub Releases 下载并缓存（仅首次需要）。
   - **纯离线/无外网电脑**：如需在完全断网或无法访问 GitHub 的电脑上运行，只需将你本机的 `C:\Users\33572\.flet\client\flet-desktop-full-0.85.1` 文件夹拷贝到目标电脑的对应用户目录下，或通过环境变量 `FLET_VIEW_PATH` 指向包含 `flet.exe` 的文件夹即可免下载直接秒开。
