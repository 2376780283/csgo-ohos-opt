# CS:GO Source 引擎 Android 移植 · Termux 构建大型 Patch 总结

> 时间跨度：从 `deps` 首次失败到 `panorama` / `panorama_text_pango` 通过  
> 环境：Termux aarch64、NDK r28c（linux-arm64 host）、API 30、不使用 distrobox  
> 目标：在 Android 设备上交叉编译 CS:GO Source 引擎（kisak-strike / CSNO 同源）并打包 APK

---

## 1. 总体结论

这次 patch 的核心工作可以分成四层：

| 层级 | 工作量 | 性质 |
|---|---|---|
| 环境与构建脚本适配 | 中 | Termux 特有的路径、工具链、符号差异 |
| 依赖补齐（proto / nut / 静态库 / 图形库） | 大 | 机械但量大，VPC 的 `custom_build_tools` 在 Termux 下生成失败 |
| V8 版本迁移（5.8 头文件 → 6.6.346 预编译库） | 最大 | 系统性 API 变化，跨一个大版本 |
| 图形库 ABI 与链接 | 中 | glib/pango/cairo/fontconfig/OpenSSL 的 32/64 位模型与 SONAME |

最终 `panorama` 和 `panorama_text_pango` 编译链接通过，`engine` 主流水线可以继续推进到 `Client_Panorama_CSGO` / `Server_CSGO`。

---

## 2. 构建流水线

```

deps → foundation → engine-deps → engine → native → package → verify → install

```

| 阶段 | 产物 | 依赖 |
|---|---|---|
| `deps` | SDL3、DXVK | NDK、CMake、Ninja、Meson |
| `foundation` | tier0/tier1/vstdlib/vtf 等 `.a` 和 `.so` | VPC、sse2neon |
| `engine-deps` | protobuf、Crypto++、mbedtls、curl、jpeg、freetype、expat、png、parsifal | NDK clang |
| `engine` | 26 个引擎模块 `.so` | 上面全部 + V8 + pango/cairo + proto/nut |
| `native` | `libmain.so` | engine 产物 |
| `package` | APK | engine-build.json + 全部 `.so` |

---

## 3. 问题分类与解决

### 3.1 环境与脚本适配（Termux 特有）

| # | 现象 | 根因 | 解决 |
|---|---|---|---|
| 1 | `shell level (1000) too high` | 脚本无条件 `exec` distrobox 包装自身 | 删除开头的 distrobox `if` 块，直接宿主环境构建 |
| 2 | `ANDROID_NDK_HOME` 设了但脚本找不到 NDK | 脚本读 `ANDROID_NDK_ROOT` | 补充 `export ANDROID_NDK_ROOT` |
| 3 | NDK host 目录是 `linux-arm64` 而非 `linux-x86_64` | Termux 是 aarch64 | `ndk_bin` 改为 `$ndk/toolchains/llvm/prebuilt/linux-arm64/bin` |
| 4 | `CMake ... api-level.h` 找不到 | NDK toolchain 把 `CMAKE_INSTALL_PREFIX` 误当 sysroot | `build_native()` 显式传 `-DCMAKE_SYSROOT` 和 `-DCMAKE_FIND_ROOT_PATH` |
| 5 | `make: /bin/bash: No such file` | VPC 生成的 makefile 硬编码 `SHELL := /bin/bash` | `make` 时传 `LINUX_TOOLS_PATH="/data/data/com.termux/files/usr/bin/"` |
| 6 | `gendbg.sh: Permission denied` | 脚本无执行位 | `chmod +x src/devtools/*.sh` |
| 7 | `tee: /tmp/...` 失败 | Termux 没有 `/tmp` | 日志写到 `$HOME/engine-build.log` |

### 3.2 VPC 构建系统

| # | 现象 | 根因 | 解决 |
|---|---|---|---|
| 8 | `unsupported option '-msse' / '-mmmx'` | VPC Makefile 硬编码 x86 选项 | 删除 `src/utils/vpc/Makefile` 中的 `-msse -mmmx` |
| 9 | `sse2neon.h not found` | 未注入 sse2neon include | `CPLUS_INCLUDE_PATH=runtime/android/deps/sse2neon` |
| 10 | `undefined symbol: glob / libiconv_open` | Termux 的 iconv/glob 符号与 GNU 命名差异 | 链接 `-liconv -landroid-glob`（`libiconv.so` 提供 `libiconv_open`） |
| 11 | `Bad Syntax in platform_dirs.vpc line:1` | `.vpc` 文件带 UTF-8 BOM | 去 BOM：`sed -i '1s/^\xEF\xBB\xBF//'`，批量处理所有 `.vpc` |
| 12 | `csgo_android_base.mak: No such file` | VPC 在 Termux 下丢失 `.mak` 扩展名 | `[[ -f csgo_android_base ]] && mv csgo_android_base csgo_android_base.mak` |
| 13 | `No rule to make target .../custom_build_tools/*.sh` | VPC 无法写入 `custom_build_tools` 脚本 | 手工创建目录 + 空壳脚本（见 §4） |

### 3.3 依赖补齐

| # | 缺失 | 处理 |
|---|---|---|
| 14 | `expat.h` / `libexpat.a` | 从 GitHub 下载 expat 2.5.0，手工交叉编译核心 `.c` 文件（绕过 CMake 4.4 的 `cmake_minimum_required` 兼容问题） |
| 15 | `libtier0_client.so` / `libvstdlib_client.so` 在 `game/bin` 而非 `src/lib/public` | 复制 `.so` 到 `src/lib/public/androidarm64/release/` |
| 16 | `libpango-1.0.so` / `libcairo.so` / `libfontconfig.so` / `libglib-2.0.so` / `libgobject-2.0.so` / `libpangoft2-1.0.so` | 从 `$PREFIX/lib` 用 `cp -L` 复制到 `runtime/android/install/lib/` |
| 17 | `libcrypto_client.a` 实为 Crypto++，无 OpenSSL AES 符号 | 见 §3.5 |

### 3.4 proto / nut 生成（`custom_build_tools`）

这是本次工作量最大的机械部分。VPC 在 Termux 下无法生成 `*.sh` 脚本，导致每个模块的 proto/nut 依赖都报 `No rule to make target`。

**通用修复模式**：

1. 用项目自带的 protobuf 2.5.0 源码编一个 aarch64 的 `protoc`（Termux 的 protobuf 36.2 与 `libprotobuf.a` 2.5.0 ABI 不兼容）
2. 用这个 `protoc` 手工生成 `.pb.cc` / `.pb.h`
3. 用 Python 把 `.nut` 转成 `_nut.h`（`static const unsigned char g_Script_xxx[]`）
4. 为每个 makefile 引用的 `custom_build_tools/*.sh` 创建 `exit 0` 空壳脚本（Debug + Release 各一份）

**各模块的依赖清单**：

| 模块 | proto | nut | 产物目录 |
|---|---|---|---|
| `gcsdk` | steammessages | — | `src/gcsdk/generated_proto/` |
| `engine` | netmessages、network_connection、engine_gcmessages | — | `src/engine/generated_proto/` |
| `panorama` | uifontfile_format | — | `src/panorama/generated_proto_panorama_text_base/` |
| `server` | netmessages、network_connection、engine_gcmessages、base_gcmessages、cstrike15_gcmessages、cstrike15_usermessages、steammessages | spawn_helper、vscript_server | `src/game/server/generated_proto/`、`obj_Server_CSGO_.../` |
| `client_panorama` | 同上 7 个 | vscript_client | `src/game/client_panorama/generated_proto/`、`src/game/client/obj_Client_Panorama_CSGO_androidarm64_client/release/` |

**关键坑**：
- `protoc` 2.5.0 要求 `-I` 和输入文件**全绝对路径**，混用相对/绝对会报 `File does not reside within any path specified`
- `cstrike15/` 子目录的 `-I` 要排在 `game/shared` **之前**，否则 canonical name 带前缀，import 对不上
- `Client_Panorama_CSGO` 的 makefile 在 `src/game/client/`，但产物路径写的是 `../client_panorama/generated_proto/`，需要手动 `mkdir -p src/game/client_panorama/generated_proto`

### 3.5 V8 5.8 → 6.6.346 迁移（核心难点）

**背景**：源码头文件是 V8 5.8.283，但可用的 Android ARM64 预编译库来自 LayaNative，版本是 6.6.346。两者跨一个大版本，API 系统性变化。

**头文件替换**：
```bash
cp -r ~/LayaNative2.0/ThirdParty/v8/include/. src/thirdparty/v8/include/
```

API 变化对照：

5.8 写法 6.6 写法 位置
v8::TryCatch tc; v8::TryCatch tc( v8::Isolate::GetCurrent() ); uijsregistration.h:239、uiengine.cpp 5 处
args.Callee()->GetName() 移除，改用固定字符串 uijsregistration.h:365/405
val->ToNumber() val->ToNumber( ctx ).ToLocalChecked() uiengine.cpp、uievent.cpp
val->ToUint32() val->ToUint32( ctx ).ToLocalChecked() uiengine.cpp:3343
v8::JSON::Parse(str) v8::JSON::Parse( ctx, str ).ToLocalChecked() uiengine.cpp:4182
v8::platform::CreateDefaultPlatform(n) v8::platform::NewDefaultPlatform(n, ...) uiengine.cpp:1525
V8::InitializeICU(path) V8::InitializeICUDefaultLocation(nullptr, path) uiengine.cpp:1535
Isolate::IdleNotification(ms) Isolate::IdleNotificationDeadline(sec) uiengine.cpp:6337/6372
V8::SetFatalErrorHandler(...) 移除，注释 uiengine.cpp:1658
#include "v8-debug.h" 移除（6.6 已删） uiengine.h:49/53
-I 缺 thirdparty/v8/include libplatform.h 里 #include "libplatform/..." 找不到 makefile INCLUDEDIRS 加 ../thirdparty/v8/include

ctx 的取法：

```cpp
v8::Isolate::GetCurrent()->GetCurrentContext()
```

风险：ToLocalChecked() 遇到非法 JS 值会 abort 进程，原代码的 if (IsEmpty()) 错误分支永远不会进。若运行时崩在 ToLocalChecked，改成：

```cpp
v8::Local<v8::Number> x;
if ( !src->ToNumber( ctx ).ToLocal( &x ) ) { /* 原有错误处理 */ }
```

3.6 图形库与 ABI

# 现象 根因 解决
18 sizeof(long) == 4 静态断言失败 项目自带的 glibconfig.h 是 32 位模型（GLIB_SIZEOF_LONG 4） 用 Termux 的 /data/data/com.termux/files/usr/lib/glib-2.0/include/glibconfig.h（GLIB_SIZEOF_LONG 8）覆盖 src/panorama_s1wrapper/thirdparty/pango/glib/glibconfig.h
19 undefined symbol: AFontMatcher_create Android 分支调用 libandroid.so 的字体 API，但 makefile 未链接 -landroid vguimatsurface 的 SystemLibraries 加 -landroid
20 undefined symbol: AES_set_decrypt_key / AES_decrypt uifontfile.cpp 用 OpenSSL 1.0.1e API，但 libcrypto_client.a 实为 Crypto++，且 src/external/openssl-1.0.1e/ 只有头文件没有源码 复制 Termux 的 libcrypto.so 到 runtime/android/install/lib/，panorama_text_pango 的 SystemLibraries 加 -lcrypto（Termux OpenSSL 3 保留了 deprecated AES_* 符号）

---

4. 可复现修复脚本

4.1 VPC 空壳脚本生成（通用）

```bash
#!/data/data/com.termux/files/usr/bin/bash
# 用法：gen_vpc_stubs <模块目录> <脚本前缀> <脚本名列表...>
gen_vpc_stubs() {
  local dir="$1"; shift
  local prefix="$1"; shift
  cd "$dir" || return 1
  for cfg in Debug Release; do
    mkdir -p "_vpc_/generated_scripts_${prefix}/androidarm64/$cfg/custom_build_tools"
    for name in "$@"; do
      local f="_vpc_/generated_scripts_${prefix}/androidarm64/$cfg/custom_build_tools/$name"
      [ -f "$f" ] || { printf '#!/data/data/com.termux/files/usr/bin/bash\nexit 0\n' > "$f"; chmod +x "$f"; }
    done
  done
}
```

4.2 proto 生成（全绝对路径）

```bash
#!/data/data/com.termux/files/usr/bin/bash
# 用法：gen_protos <输出目录> <proto 目录1> <proto 文件1> ...
gen_protos() {
  local out="$1"; shift
  local protoc="$HOME/csgo-ohos-opt/CSGO-Source-Linux-20260928/src/thirdparty/protobuf-2.5.0/src/protoc"
  local pbsrc="$HOME/csgo-ohos-opt/CSGO-Source-Linux-20260928/src/thirdparty/protobuf-2.5.0/src"
  mkdir -p "$out"
  local -a incs
  while [ "$#" -gt 0 ] && [ -d "$1" ]; do incs+=("-I$1"); shift; done
  "$protoc" "${incs[@]}" -I"$pbsrc" --cpp_out="$out" "$@"
}
```

4.3 nut → h（Python）

```python
from pathlib import Path
def nut_to_h(nut_path, out_path, symbol):
    data = Path(nut_path).read_bytes()
    lines = [f"// Auto-generated from {nut_path}",
             f"static const unsigned char {symbol}[] = {{"]
    for i in range(0, len(data), 16):
        lines.append("    " + ", ".join(f"0x{b:02x}" for b in data[i:i+16]) + ",")
    lines.append("    0x00")
    lines.append("};")
    Path(out_path).write_text("\n".join(lines) + "\n")
```

---

5. 关键改动文件清单

文件 改动
scripts/build-android.sh 删除 distrobox 块；build_native() 加 CMAKE_SYSROOT / CMAKE_FIND_ROOT_PATH；build_engine() / build_foundation() 加 .mak 重命名和 LINUX_TOOLS_PATH
android/CMakeLists.txt 变量名 OHOS → Android；__OHOS__ → ANDROID；hilog_ndk.z → log；.so import 路径改 game/bin/androidarm64/release/
android/native/android_main.cpp 1 处 SDL_GetOpenHarmonyInternalStoragePath() → PLATFORM_INTERNAL_STORAGE
src/utils/vpc/Makefile 删除 -msse -mmmx；LINK 加 -liconv -landroid-glob
src/thirdparty/v8/include/ 整体替换为 LayaNative 的 V8 6.6.346 头文件
src/panorama/uiengine.h 注释 v8-debug.h 两行 include
src/panorama/uiengine.cpp 8 处 V8 6.6 API 适配（见 §3.5）
src/panorama/uievent.cpp 7 处 ToNumber() 加 context
src/public/panorama/uijsregistration.h TryCatch 加 isolate；Callee() 移除
src/panorama_s1wrapper/thirdparty/pango/glib/glibconfig.h 用 Termux aarch64 版本覆盖
src/panorama/panorama_androidarm64.mak INCLUDEDIRS 加 ../thirdparty/v8/include
src/panorama/panorama_text_pango_androidarm64.mak SystemLibraries 加 -lcrypto
src/vguimatsurface/vguimatsurface_androidarm64.mak SystemLibraries 加 -landroid
src/panorama_text_pango/uitextlayoutpango.cpp 无需改，靠 glibconfig.h 替换解决
src/game/client_panorama/generated_proto/ 新建，7 组 .pb.cc/.pb.h
src/game/client/obj_Client_Panorama_CSGO_androidarm64_client/release/ 新建，vscript_client_nut.h
src/game/client/_vpc_/generated_scripts_client_panorama_csgo/ 16 个空壳脚本
src/game/server/generated_proto/ 7 组 .pb.cc/.pb.h
src/game/server/obj_server_androidarm64_client/release/ spawn_helper_nut.h、vscript_server_nut.h
src/game/server/_vpc_/generated_scripts_server_csgo/ 18 个空壳脚本
src/engine/generated_proto/ 3 组 .pb.cc/.pb.h
src/engine/_vpc_/generated_scripts_engine/ 6 个空壳脚本
src/gcsdk/generated_proto/ steammessages.pb.cc/.pb.h
src/gcsdk/_vpc_/generated_scripts_gcsdk/ 4 个空壳脚本
runtime/android/install/lib/ 新增 libglib-2.0.so、libgobject-2.0.so、libpangoft2-1.0.so、libcrypto.so、libssl.so（来自 Termux）
src/lib/common/androidarm64/ 新增 libv8*.cr.so（LayaNative 6.6.346）、libcrypto_client.a（Crypto++）、libparsifal.a、bzip2_client.a、havana_constraints_client.a、hk_base_client.a、hk_math_client.a、ivp_compactbuilder_client.a、ivp_physics_client.a、jpeglib_client.a

---

6. 遗留问题与风险

6.1 高风险

1. V8 6.6 与源码头文件版本差异
   · 源码头文件是 5.8，库是 6.6，这次靠 LayaNative 的 6.6 头文件 + patch 调用点强行对齐
   · 未来若升级 V8，必须重新对齐头文件和库
2. Termux 的 glib/pango/cairo/OpenSSL 是 API 24 构建
   · 项目目标是 API 30，链接能过，但运行时可能因 API 24 → 30 的符号差异崩溃
   · 建议：最终发布前，在 x86_64 主机上用 NDK r28c 按 API 30 重新交叉编译这些库
3. ToLocalChecked() 的运行时 abort 风险
   · uiengine.cpp / uievent.cpp 共 11 处改成 ToLocalChecked()
   · 若 JS 传入非法值，进程会 abort
   · 建议：真机跑通后，逐步改成 ToLocal(&local) + 原有错误处理
4. libcrypto.so 的 SONAME 是 libcrypto.so.3
   · APK 打包时若只带 libcrypto.so，运行时 linker 找不到 libcrypto.so.3
   · 需在 package_apk 阶段补一份 libcrypto.so.3

6.2 中风险

5. panorama 的 V8 调用点可能还有遗漏
   · 这次只暴露了 8 处，uiengine.cpp 有 6000+ 行，后续编译可能还有 MaybeLocal 转换、Context::New 签名等
6. custom_build_tools 空壳脚本只解决“脚本存在”
   · .pb.cc / _nut.h 的真实产物靠手工生成，脚本本身是 exit 0
   · 如果 VPC 重新生成 makefile（比如改了 .vpc），空壳和产物都要重做
7. android-build-tools.py 的 stage() 硬编码 linux-x86_64
   · runtime = ndk / "toolchains/llvm/prebuilt/linux-x86_64/..." 在 Termux 上找不到
   · 需改成动态 host tag 检测

6.3 低风险

8. bzip2_client.a 等 .a 的路径不一致
   · src/lib/common/androidarm64/release/ 和顶层各有一份，需保持同步
   · 建议：在 foundation / engine 后加一步 rsync 把所有 .a 平铺到顶层

---

7. 后续建议

7.1 短期（让 engine 完整跑通）

1. 继续跑 make，逐个修复 Client_Panorama_CSGO / Server_CSGO 剩余的 proto/nut/.a 缺失
2. 修 android-build-tools.py 的 linux-x86_64 硬编码
3. 补 libcrypto.so.3 到 APK 打包清单
4. 跑 native + package + verify，打出第一个完整 APK

7.2 中期（提升可维护性）

5. 把这次所有手工 patch 整理成 android/patches/termux-*.patch，在 build-android.sh 里自动应用
6. 把 proto/nut/空壳脚本的生成逻辑写进 scripts/android-build-tools.py，替代手工 Python
7. 把 V8 6.6 的 API 适配封装成 v8_compat.h 里的 inline 辅助函数（V8ToNumber(isolate, val) 等），减少重复 patch

7.3 长期（生产级构建）

8. V8 换回源码编译：在 x86_64 主机上按 DEPS 的 commit 编 V8 5.8.283，彻底消除版本差异
9. 图形库按 API 30 交叉编译：glib/pango/cairo/fontconfig 用 NDK r28c 重编，替代 Termux 的 API 24 版本
10. CI 矩阵：x86_64 构建重型依赖 → Termux 只做 native / package / install

---

8. 一句话总结

这次 patch 的本质是在 Termux 这个“半个 Linux”环境里，把一个为 x86_64 Linux 设计的构建系统（VPC + Make）和一个为 V8 5.8 设计的源码，硬接到 V8 6.6 的 ARM64 预编译库上。
所有问题都可以归为三类：路径假设（/bin/bash、/tmp、linux-x86_64）、版本假设（V8 5.8 vs 6.6、protobuf 2.5 vs 36、glib 32 vs 64 位）、产物假设（.mak 扩展名、custom_build_tools 脚本、.so/.a 路径）。
逐类击破后，engine 主流水线可以继续推进。
