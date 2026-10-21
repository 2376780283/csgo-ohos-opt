# [Feature] 构建与依赖问题自动化诊断：Build Doctor + 可复现构建报告

> 建议标签：`build`、`android`、`termux`、`automation`、`dx`、`dependencies`  
> 建议优先级：`P1`

## 摘要

本项目在 Termux/aarch64 + Android NDK 环境下推进 CS:GO Source 引擎 Android 移植时，构建链路反复被环境差异、依赖缺失、VPC/GN 工具链兼容性、路径硬编码、文件编码和产物命名不一致阻塞。

大量问题具有高度重复模式：现象相似、根因分散、修复手段手工且易丢失。

建议新增一套 **Build Doctor / 构建自动化诊断与依赖供给机制**，覆盖：

- 环境预检与宿主能力判定
- 依赖清单、版本锁定与产物路径校验
- Termux 专用 patch 自动应用与幂等修复
- VPC/GN/CMake 常见错误自动识别与建议
- 构建失败时自动生成 Markdown Issue 草稿
- x86_64 远程构建 V8/Pango/Cairo 产物回传流程

目标是让“踩坑—定位—修复—回归”从人工经验变成可执行脚本和可复现报告。

---

## 环境

- 宿主：Termux，aarch64 Android
- NDK：`android-ndk-r28c`，`toolchains/llvm/prebuilt/linux-arm64`
- SDK：`build-tools/36.1.0`，`platforms/android-36`
- 构建脚本：`scripts/build-android.sh`
- 关键阶段：`deps`、`foundation`、`v8`、`engine`、`native`、`package`、`verify`
- 不使用 distrobox，直接在 Termux 宿主环境构建
- 目标：产出可安装 APK，并逐步补齐引擎模块

---

## 目标

1. 新环境首次构建时，能明确告诉开发者“哪些能本地构建，哪些必须远程构建”。
2. 依赖缺失时，输出精确的仓库 URL、commit、目标路径和修复命令。
3. 常见错误自动归类，并给出可执行修复建议。
4. Termux 专用修复不依赖人工记忆，不因 `git checkout` 或重跑脚本丢失。
5. 构建失败时自动生成可提交的 Issue 报告，包含环境、日志尾部、错误分类和已尝试步骤。
6. x86_64 主机可一键构建 V8/Pango/Cairo 等重型依赖，Termux 端只负责 `native`、`package`、`install`。

---

## 已遇到的问题与解决过程沉淀

| # | 阶段 | 现象 | 根因 | 已采用解决 | 自动化需求 |
|---|---|---|---|---|---|
| 1 | 启动 | `shell level (1000) too high`，大量 bash 自调用 | 脚本无条件 `exec` 自身或 distrobox 包装导致递归 | 删除/绕过 distrobox 自调用块 | `doctor` 检测脚本自调用与 `CONTAINER_ID` 逻辑 |
| 2 | 环境 | `ANDROID_NDK_HOME` 已设但脚本找不到 NDK | 脚本读取 `ANDROID_NDK_ROOT` | 补充 `ANDROID_NDK_ROOT` | 环境变量别名检查与提示 |
| 3 | CMake | `find_package(SDL3)` 失败 | `CSGO_ANDROID_DEPS` 为空或缓存残留 | 改 CMakeLists 变量名、清 `build/` | 依赖前缀与 CMakeCache 校验 |
| 4 | DXVK | `Subproject exists but has no meson.build` | `libdisplay-info` 子项目空壳 | 删除空目录后重新 Meson | 子模块完整性检查 |
| 5 | DXVK | `fatal error: 'd3d9.h' file not found` | `include/native/directx` 子模块未 checkout | `git submodule update --init --recursive --force` | 子模块工作树非空校验 |
| 6 | VPC | `unsupported option '-msse' / '-mmmx'` | VPC Makefile 硬编码 x86 选项 | 删除 x86 选项 | Termux patch 自动应用 |
| 7 | VPC | `fatal error: 'sse2neon.h' file not found` | 未注入 sse2neon include | `CPLUS_INCLUDE_PATH` 或 Makefile 加 `-I` | 自动注入并校验 |
| 8 | VPC | `undefined symbol: glob / libiconv_open` | Termux 库符号与 GNU 命名差异 | 链接 `-liconv -landroid-glob` | 依赖符号探测与链接参数生成 |
| 9 | VPC | `Bad Syntax in platform_dirs.vpc line:1` | `.vpc` 文件带 UTF-8 BOM | 去 BOM | 批量 BOM/CRLF 检查 |
| 10 | VPC | `make: csgo_android_base.mak: No such file` | VPC 输出丢失 `.mak` 扩展名 | 手工重命名 | 自动重命名 master makefile |
| 11 | make | `make: /bin/bash: No such file` | Makefile 硬编码 `SHELL := /bin/bash` | 传 `LINUX_TOOLS_PATH` | 自动注入 Termux bash 路径 |
| 12 | vstdlib | `No rule to make target .../libtier0_client.so` | `.so` 实际在 `game/bin`，未同步到 `src/lib/public` | 复制 `.so` 到期望目录 | 产物路径矩阵与自动同步 |
| 13 | vstdlib | `gendbg.sh: Permission denied` | 脚本无可执行权限 | `chmod +x` | 可执行位检查 |
| 14 | native | `use of undeclared identifier 'SDL_GetOpenHarmonyInternalStoragePath'` | OHOS 专有调用未包 `#ifdef` | 改用 `PLATFORM_INTERNAL_STORAGE` | 平台 API 扫描 |
| 15 | expat | `fatal error: 'expat.h' file not found` | Expat 未交叉编译 | 手工交叉编译并安装到 prefix | 依赖清单补全 |
| 16 | CMake | `api-level.h` 找不到 | CMake/NDK 推导 sysroot 错误 | 显式传 `CMAKE_SYSROOT` / `CMAKE_FIND_ROOT_PATH` | 工具链参数自动注入 |
| 17 | stage | 缺 `engine-build.json` | `engine` 未完成或未生成 receipt | 需先完成 engine 或生成记录 | 构建阶段依赖图与前置检查 |
| 18 | V8 | `python3_bin_reldir.txt not found`、`gclient` 卡住、GN 报 `Assignment had no effect`、`third_party/simdutf` 缺失 | Termux 上 V8/depot_tools/GN 兼容性极差 | 手工补文件、改 GN 参数、补子模块 | 宿主能力判定：V8 建议远程构建 |
| 19 | 依赖 | `libpango-1.0.so` / `libcairo.so` 缺失 | 不在 `build_engine_deps()` 自动列表 | 临时从 Termux 包复制 | 依赖矩阵与 ABI 风险提示 |
| 20 | 依赖 | `libcrypto_client.a` 缺失 | Crypto++ 产物名/路径不一致 | 复制/重命名 `libcryptopp.a` | 产物命名映射 |

---

## 建议的自动化功能设计

### 1. `scripts/doctor.sh`：环境与能力预检

检查并输出：

- 宿主架构、OS、是否 Termux
- NDK host tag 是否存在：`linux-arm64` / `linux-aarch64` / `linux-x86_64`
- NDK sysroot 是否含 `android/api-level.h`
- SDK `build-tools`、`platforms/android-*.jar` 是否存在
- `cmake`、`ninja`、`meson`、`clang`、`python3`、`python2`、`gn`、`rustc`
- 磁盘空间、内存建议、`BUILD_JOBS` 建议
- 环境变量污染：`LDFLAGS`、`CPLUS_INCLUDE_PATH`、`CMAKE_*`
- 关键子模块工作树是否为空
- `.vpc` 文件是否含 BOM/CRLF
- 脚本可执行位：`gendbg.sh`、`devtools/*.sh`
- 明确结论：`V8 本地构建可行性：否，建议 x86_64 远程构建`

### 2. `scripts/deps.lock.json`：依赖清单与产物矩阵

为每个依赖记录：

```json
{
  "name": "sse2neon",
  "url": "...",
  "commit": "...",
  "install": ["include/sse2neon.h"],
  "required_by": ["foundation", "engine"],
  "host_supported": ["x86_64", "aarch64"]
}
```

覆盖：SDL3、DXVK、sse2neon、mbedtls、curl、jpeg、freetype、expat、protobuf、Crypto++、Pango、Cairo、V8。

3. scripts/apply-termux-patches.sh：幂等补丁

自动处理：

· VPC Makefile 删除 -msse -mmmx
· 注入 -liconv -landroid-glob
· 注入 CPLUS_INCLUDE_PATH 或 -I sse2neon
· 批量去 BOM / CRLF
· 自动重命名 csgo_android_base → .mak、csgo_android_engine → .mak
· 自动注入 LINUX_TOOLS_PATH="$PREFIX/bin/"
· 修正 android-build-tools.py 中 linux-x86_64 硬编码

要求：可重复执行，已应用则跳过，不覆盖用户本地修改。

4. scripts/build-stage.sh：阶段包装器

统一封装 foundation / engine / native：

· 清理指定 CMake 缓存
· 设置阶段专用环境变量
· 自动检测缺失 .mak 并重命名
· 失败时自动降并发重试一次
· 日志写入 runtime/android/logs/<stage>-<timestamp>.log
· 输出“下一步建议”

5. 错误分类与建议引擎

根据日志关键字自动分类：

关键字 分类 建议
bad interpreter: No such file shebang termux-fix-shebang / termux-exec
Assignment had no effect GN 版本 升级 GN 或注释变量
No rule to make target '*.mak' VPC 扩展名 自动重命名
'/bin/bash': No such file SHELL 注入 LINUX_TOOLS_PATH
sse2neon.h not found include 注入 sse2neon 路径
undefined symbol: glob/libiconv Termux 符号 链接 -liconv -landroid-glob
api-level.h sysroot 显式 CMAKE_SYSROOT
Subproject ... no meson.build 子模块空壳 删除后重新 sync
engine-build.json 阶段顺序 先跑 engine 或生成 receipt

6. scripts/build-report.sh：失败自动生成 Issue 草稿

失败时生成 Markdown：

· 环境摘要
· git 状态与关键文件 SHA
· 最后 100 行日志
· 错误分类与建议
· 已应用 patch 列表
· 缺失依赖与期望路径
· 可选：生成 GitHub Issue URL 参数

注意：默认不上传日志，仅本地生成。

7. 远程构建 V8/Pango/Cairo

新增文档与脚本：

· scripts/remote-build-v8.sh：在 x86_64 Linux 上跑 gclient sync、gn gen、ninja
· 产物清单：libv8.cr.so、libv8_libbase.cr.so、libv8_libplatform.cr.so
· libpango-1.0.so、libcairo.so 及依赖
· scripts/import-remote-artifacts.sh：校验 ABI、SONAME、Build ID 后放入 Termux 项目目录

---

实现里程碑

☐ M1：doctor.sh + 错误分类表
☐ M2：deps.lock.json + 子模块/依赖完整性检查
☐ M3：Termux 幂等 patch 集
☐ M4：build-stage.sh 自动重命名与 LINUX_TOOLS_PATH
☐ M5：build-report.sh 生成 Issue 草稿
☐ M6：x86_64 远程构建 V8/Pango/Cairo 文档与产物导入脚本
☐ M7：CI 矩阵：x86_64 构建重型依赖，Termux 验证 native/package/install

---

验收标准

☐ 在全新 Termux/aarch64 环境运行 doctor.sh，能明确报告 V8 不建议本地构建，并列出缺失工具与依赖。
☐ foundation 阶段自动处理 VPC -msse/-mmmx、sse2neon.h、glob/libiconv、BOM、.mak 扩展名和 /bin/bash，无需手工改文件。
☐ engine 阶段缺失依赖时，输出精确 URL、commit、目标路径和复制/编译命令。
☐ native 阶段自动检测 CMAKE_SYSROOT 和平台 API 宏问题，失败时给出明确建议。
☐ 任一阶段失败后，build-report.sh 生成可提交的 Markdown Issue 草稿。
☐ x86_64 远程构建产物可一键导入 Termux，并通过 verify 的 ELF/16KiB/Build ID 检查。
☐ 所有 Termux 专用修复可重复执行，不因 git checkout 或重跑脚本丢失。

---

风险与边界

· V8 在 Termux/aarch64 本地构建成本极高，自动化应优先推荐 x86_64 远程构建，而不是鼓励硬啃。
· 从 Termux 本地包复制的 libpango-1.0.so / libcairo.so 存在 ABI 与 API level 风险，只能作为临时验证，不应作为正式发布依赖。
· 预编译 libv8.cr.so 必须记录来源与版本；版本不匹配可能导致链接失败或运行时崩溃。
· 自动 patch 必须幂等，且不能覆盖用户本地修改；建议使用 git apply --reverse --check 模式。
· 构建报告默认本地生成，避免日志中路径或环境信息被自动上传。

---

附录：建议新增文件

```text
scripts/doctor.sh
scripts/apply-termux-patches.sh
scripts/build-stage.sh
scripts/build-report.sh
scripts/deps.lock.json
scripts/import-remote-artifacts.sh
docs/termux-build-doctor.md
docs/remote-v8-pango-cairo-build.md
```