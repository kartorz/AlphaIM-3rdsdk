# AlphaIM-plugin

为使用**私有（自带）Qt/GTK** 的应用编译 AlphaIM 输入法插件。

AlphaIM 是一款 Linux 平台的离线智能拼音输入法（GPLv3），通过三种协议接入应用：

- XIM
- GTK 输入法模块（immodule）
- Qt platform input context 插件

使用发行版 Qt/GTK 的应用，安装 AlphaIM 官方包即可。但有些应用自带私有构建的 Qt/GTK：WPS Office 内置了 `kso_qt` 命名空间的 Qt 5.12.12；部分 IM、Electron 类应用静态链接或自带私有 GTK。这类应用不会加载系统插件——命名空间、ABI、模块搜索路径都对不上。本仓库保存针对这些私有库编译插件所需的构建描述，以及编译用的 Qt SDK 头文件和工具。

> 项目描述中的“似有”指“私有”，即应用自带、非系统提供的 Qt/GTK。

## 当前状态

- **WPS Office**（私有 Qt 5.12.12，`kso_qt` 命名空间）：已收录，位于 `wps/Qt5.12/`。
- **其他私有 Qt 应用**：可按同样的模式扩展，见[适配其他私有 Qt 应用](#适配其他私有-qt-应用)。
- **私有 GTK 应用**：尚未收录构建描述，思路见[私有 GTK](#私有-gtk)。

## 目录结构

```
.
└── wps/Qt5.12/
    ├── Makefile                          # 针对 WPS 私有 Qt 的构建与安装规则
    ├── bin/                              # Qt 5.12.12 官方 SDK 的构建工具（moc、qmake 等）
    └── include/                          # Qt 5.12.12 官方 SDK 头文件（含 qpa 私有头）
```

这里放的是 **Qt 官方 5.12.12 SDK** 的头文件和工具，不是 WPS 的库；链接时使用的是 WPS 安装目录里的 `libQt5*Kso.so`。仓库只保留了 SDK 的 `include/` 和 `bin/`，没有 `lib/`，构建过程也不依赖 SDK 的库目录。

## 工作原理

Qt 插件实现在 AlphaIM 源码的 `src/protocol/qt/`：

- `main.cpp` 注册 `QPlatformInputContextPlugin`，插件键为 `alphaim`（见 `aim.json`）。
- 插件元数据 IID 为 `org.qt-project.Qt.QPlatformInputContextFactoryInterface.5.1`。
- 输入上下文通过 D-Bus 与 `alphaimd` 通信，接口为 `org.freedesktop.AlphaIM` 和 `org.freedesktop.AlphaIM.InputContext`。

WPS 的插件必须按 WPS 私有 Qt 的构建方式编译，关键定义在 `wps/Qt5.12/Makefile`：

- `-DQT_NAMESPACE=kso_qt`：WPS 的 Qt 在 `kso_qt` 命名空间内编译，插件引用的所有 Qt 类型和符号都要带这个命名空间。
- `-DAIM_WPS_QT`：适配 WPS 私有 Qt 与上游 Qt 5.12 的 API 差异（`reset(bool)` 签名、用 `QInputMethod::queryFocusObject` 查询光标矩形）。
- `-DQT_PLUGIN`：导出插件元数据。
- 链接 `libQt5GuiKso`、`libQt5DBusKso`、`libQt5CoreKso`，rpath 为 `$ORIGIN/../../lib:$(WPS_ROOT)`。

## 前置条件

- Linux x86_64
- `g++`（支持 `-std=gnu++11`）
- AlphaIM 源码，且存在 `src/protocol/qt/` 目录
- 已安装 WPS Office（默认路径 `/opt/kingsoft/wps-office/office6`）
- `libGL`、`pthread`（链接参数 `-lGL -lpthread`）
- 安装插件需要写 WPS 安装目录的权限（通常用 `sudo`）

本仓库自带 Qt 5.12.12 的 `moc` 和头文件，不需要另外安装 Qt SDK。

## 快速开始（WPS）

### 1. 准备 AlphaIM 源码

Makefile 里的 `AlphaIM` 变量指定 **AlphaIM 源码的位置**，默认 `../../../AlphaIM`（相对 `wps/Qt5.12/`，也就是与本仓库同级的 `AlphaIM/` 目录）：

```bash
git clone <AlphaIM 仓库地址> ../AlphaIM
```

如果源码在别处，构建时用 `AlphaIM=` 指定绝对路径即可，不必移动它。

### 2. 确认 WPS 路径

Makefile 里的 `WPS_ROOT` 变量指定 **WPS 的安装目录**（即 `office6` 目录），默认 `/opt/kingsoft/wps-office/office6`。WPS 装在别处时覆盖它：

```bash
make WPS_ROOT=/path/to/wps-office/office6 ...
```

Makefile 的 `check` 目标会验证 WPS 目录和 `libQt5GuiKso.so.5.12.12`、`libQt5DBusKso.so.5.12.12`、`libQt5CoreKso.so.5.12.12` 是否存在。

### 3. 构建

建议出树构建，避免在仓库目录里留下 `.o`、`.so` 和 moc 生成的中间文件：

```bash
cd wps/Qt5.12
make BUILD_DIR=/tmp/alphaim-wps-build AlphaIM=/path/to/AlphaIM
```

如果 AlphaIM 就在默认的 `../../../AlphaIM`，可以简化为 `make BUILD_DIR=/tmp/alphaim-wps-build`。

产物：`/tmp/alphaim-wps-build/libalphaimplatforminputcontextplugin.so`。

### 4. 安装

```bash
sudo make install BUILD_DIR=/tmp/alphaim-wps-build AlphaIM=/path/to/AlphaIM
```

插件会被安装到 `$(WPS_ROOT)/qt/plugins/platforminputcontexts/`。也可以用 `WPS_PLUGIN_DIR=` 指定其他目录。

默认的 `WPS_ROOT` 是 `/opt/kingsoft/wps-office/office6`，安装路径即 `/opt/kingsoft/wps-office/office6/qt/plugins/platforminputcontexts/`。

### 5. 启用

- 确保 `alphaimd` 正在运行（`im-config` 选择 AlphaIM 后会自动启动，也可手动 `alphaimd &`）。
- 确保 WPS 进程能看到 `QT_IM_MODULE=alphaim`。`im-config` 选择 AlphaIM 后会话环境里已有该变量；单独启动时可以用：

```bash
QT_IM_MODULE=alphaim wps
```

然后重启 WPS。

## Makefile 变量

- `AlphaIM`：**指定 AlphaIM 源码的位置**，即 AlphaIM 源码根目录（其中应有 `src/protocol/qt/`）。默认 `../../../AlphaIM`，相对 `wps/Qt5.12/`，指向与本仓库同级的 `AlphaIM/`。源码不在那里时必须显式覆盖，例如 `make AlphaIM=/home/you/src/AlphaIM`。
- `WPS_ROOT`：**指定 WPS 的安装目录**，即 WPS 的 `office6` 目录，插件链接的 `libQt5*Kso.so` 和默认安装位置都基于它。默认 `/opt/kingsoft/wps-office/office6`，WPS 装在别处时用 `make WPS_ROOT=/path/to/office6` 覆盖。
- `WPS_PLUGIN_DIR`：插件安装目录，默认 `$(WPS_ROOT)/qt/plugins/platforminputcontexts`。
- `QTMAKE`、`MOC`：Qt 构建工具，默认取本目录的 `bin/qmake`、`bin/moc`。
- `BUILD_DIR`：中间产物和产物的输出目录，默认 `.`（当前目录）。
- `CXX`、`CXXFLAGS`、`INSTALL`：编译器、编译参数、安装命令，可按需覆盖。

注意 `BUILD_DIR` 默认为当前目录：此时 `make clean` 执行的是 `rm -rf .`（GNU rm 会拒绝删除 `.`），并不能清理默认构建产物；构建还会在 `wps/Qt5.12/` 里生成 `moc_qaim*.cpp`、`main.moc` 和 `.o` 文件。因此推荐始终显式指定 `BUILD_DIR`，清理时也用同一个值：

```bash
make clean BUILD_DIR=/tmp/alphaim-wps-build
```

## 验证

检查动态库解析和插件元数据：

```bash
ldd /tmp/alphaim-wps-build/libalphaimplatforminputcontextplugin.so | grep Kso
readelf -d /tmp/alphaim-wps-build/libalphaimplatforminputcontextplugin.so | grep -E 'RPATH|RUNPATH'
strings /tmp/alphaim-wps-build/libalphaimplatforminputcontextplugin.so \
  | grep -E 'alphaim|QPlatformInputContextFactoryInterface'
```

预期结果：

- 三个 Kso 库都解析到 WPS 安装目录（`/opt/kingsoft/wps-office/office6/libQt5*Kso.so.5`）。
- RPATH 含 `$ORIGIN/../../lib` 和 WPS 目录。
- 字符串中包含 `alphaim` 和 `org.qt-project.Qt.QPlatformInputContextFactoryInterface.5.1`。

排查 WPS 的插件加载过程：

```bash
QT_DEBUG_PLUGINS=1 QT_IM_MODULE=alphaim wps
```

输出里应能看到从 `platforminputcontexts` 目录加载 `libalphaimplatforminputcontextplugin.so`，并创建 `alphaim` 输入上下文。若插件加载失败，通常先检查是不是版本、命名空间或依赖库对不上。

## 适配其他私有 Qt 应用

前提：应用自带的 Qt 与官方 Qt 的 ABI 兼容，只是命名空间或库名不同。静态链接私有 Qt 的应用无法用插件方式接入（见[已知限制](#已知限制)）。

1. **确认 Qt 版本和命名空间**。查看应用自带的 `libQt5Core.so*`：
   - 版本：`strings <libQt5Core.so> | grep -m1 'Qt 5\.'`
   - 命名空间：`nm -D --defined-only <libQt5Core.so> | grep 'QCoreApplication'`，看符号是否带命名空间前缀（例如 `kso_qt`）。没有前缀说明使用默认命名空间，应去掉 `-DQT_NAMESPACE=...`。

2. **准备头文件**。取与目标 Qt 完全同版本的官方 SDK，把 `include/` 和 `bin/`（至少要 `moc`）放到新的目录，例如 `myapp/Qt5.15/`。`moc` 的版本必须与目标 Qt 一致，私有头 `qpa/qplatforminputcontext.h` 也要在 include 路径里。

3. **复制并修改 Makefile**：
   - 把 `QT_NAMESPACE` 改为目标命名空间，或删除该定义。
   - 去掉 `AIM_WPS_QT`，除非目标 Qt 有同样的 API 差异。
   - 把 `WPS_LIBS` 改成目标应用的 Gui/DBus/Core 库及其路径。
   - 把 rpath 和 `WPS_PLUGIN_DIR` 改成该应用的 `platforminputcontexts` 目录。
   - 保留 `-DQT_PLUGIN`、`-fPIC`、`-shared`。

4. **构建并验证**，方法同上。

5. **让应用加载插件**：把插件放到应用自己的 `platforminputcontexts` 目录，并设置 `QT_IM_MODULE=alphaim`。若应用不读取系统环境或插件目录，可以用
   `QT_PLUGIN_PATH=<插件目录> QT_IM_MODULE=alphaim <app>` 启动。

## 私有 GTK

GTK 的输入法模块源码是 AlphaIM 的 `src/protocol/gtk/gaimimcontext.c`，模块名为 `im-alpha.so`（GTK 4 为 `libim-alpha.so`），安装位置为：

```
<libdir>/gtk-2.0/<binary-version>/immodules/
<libdir>/gtk-3.0/<binary-version>/immodules/
<libdir>/gtk-4.0/<binary-version>/immodules/
```

对自带私有 GTK 的应用，需要用**该应用所用 GTK 版本**的头文件和库重新编译 immodule，并让应用从对应目录加载：

```bash
GTK_IM_MODULE=im-alpha GTK_PATH=<包含 gtk-3.0/... 的前缀> <app>
```

GTK 会校验模块的 `gtk_binary_version`，版本不一致时模块会被忽略，这一点比 Qt 更严格。本仓库目前只包含 WPS/Qt 的构建描述，尚未加入私有 GTK 的构建文件。

## 故障排查

- **WPS 里输入法没反应**：确认 `alphaimd` 在运行、`QT_IM_MODULE=alphaim`、插件在 `$(WPS_ROOT)/qt/plugins/platforminputcontexts/`，然后用 `QT_DEBUG_PLUGINS=1` 看加载日志。
- **插件加载时报 undefined symbol**：`QT_NAMESPACE` 与目标 Qt 不一致，或链接到了错误的库。检查 `ldd` 和 `nm -D` 的符号前缀。
- **`make check` 报 WPS 库缺失**：`WPS_ROOT` 不对，或 WPS 版本变化导致库名不再是 `libQt5*Kso.so.5.12.12`。
- **moc 报错或生成代码不兼容**：`moc` 版本必须与目标 Qt 完全一致，不能混用系统 Qt 的 `moc`。
- **WPS 升级后插件失效**：WPS 可能更换了 Qt 版本或库布局。重新确认版本、更新 SDK 头文件后重编。

## 已知限制

- 当前只提供 x86_64 的构建配置。
- 插件必须与目标应用的 Qt/GTK 版本、命名空间、编译器 ABI 匹配，不能跨版本通用。
- 静态编译私有 Qt 的应用（例如部分 Linux 微信版本）无法通过插件接入，可改用托盘菜单的“输入并复制”小窗，再粘贴到应用中。
- 本仓库的 Qt 头文件来自官方 5.12.12 SDK，仅用于编译；不包含 WPS 的任何库或二进制。

## 许可

- AlphaIM 本体：GPLv3。
- 插件源码：同 AlphaIM。
- 本仓库内的 Qt 头文件和工具：遵循 Qt 自身的许可条款，见各文件头部声明。
