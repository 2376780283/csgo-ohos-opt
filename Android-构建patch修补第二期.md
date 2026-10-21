# CS:GO Source 引擎 Android 移植 · Termux 构建全过程总结

> 环境：Termux aarch64 (Android) + NDK r28c (`linux-arm64` host) + API 30  
> 目标：不依赖 distrobox，在 Termux 宿主环境直接交叉编译 CS:GO Source 引擎（kisak-strike），产出可安装 APK  
> 起点：`build-android.sh build` 无限递归  
> 终点：`panorama` 及依赖模块编译链接通过，`engine` 主流水线推进到 `Client_Panorama_CSGO` / `Server_CSGO`

---

## 1. 总体结论

这次移植的本质是：**在一个"半个 Linux"的 Termux 环境里，把一个为 x86_64 Linux 设计的构建系统（VPC + GNU Make），和一个为 V8 5.8 设计的源码，硬接到 V8 6.6 的 ARM64 预编译库上。**

所有问题都可以归为三类：

| 类型 | 典型表现 |
|---|---|
| **路径假设** | `/bin/bash`、`/tmp`、`linux-x86_64`、`$(CFG)` 未展开 |
| **版本假设** | V8 5.8 vs 6.6、protobuf 2.5 vs 36、glib 32 位 vs 64 位 |
| **产物假设** | `.mak` 扩展名丢失、`custom_build_tools` 脚本不存在、`.so`/`.a` 路径不一致 |

逐类击破后，`engine` 主流水线可以继续推进。

---

## 2. 构建流水线

```

deps → foundation → engine-deps → engine → native → package → verify → install

```

| 阶段 | 产物 | 关键依赖 |
|---|---|---|
| `deps` | SDL3、DXVK | NDK、CMake、Ninja、Meson |
| `foundation` | tier0/tier1/vstdlib/vtf 等 `.a` 和 `.so` | VPC、sse2neon |
| `engine-deps` | protobuf、Crypto++、mbedtls、curl、jpeg、freetype、expat、png、parsifal | NDK clang |
| `engine` | 26 个引擎模块 `.so` | 上面全部 + V8 + pango/cairo + proto/nut |
| `native` | `libmain.so` | engine 产物 |
| `package` | APK | `engine-build.json` + 全部 `.so` |

---

## 3. 问题分类与解决

### 3.1 环境与脚本适配（Termux 特有）

| # | 现象 | 根因 | 解决 |
|---|---|---|---|
| 1 | `shell level (1000) too high` | 脚本无条件 `exec` distrobox 包装自身 | 删除 `build-android.sh` 开头的 distrobox `if` 块 |
| 2 | `ANDROID_NDK_HOME` 已设但脚本找不到 NDK | 脚本读 `ANDROID_NDK_ROOT` | `export ANDROID_NDK_ROOT` |
| 3 | NDK host 目录是 `linux-arm64` 而非 `linux-x86_64` | Termux 是 aarch64 | `ndk_bin=$ndk/toolchains/llvm/prebuilt/linux-arm64/bin` |
| 4 | `CMake ... api-level.h` 找不到 | NDK toolchain 把 `CMAKE_INSTALL_PREFIX` 误当 sysroot | `build_native()` 显式传 `-DCMAKE_SYSROOT` / `-DCMAKE_FIND_ROOT_PATH` |
| 5 | `make: /bin/bash: No such file` | makefile 硬编码 `SHELL := /bin/bash` | `make` 时传 `LINUX_TOOLS_PATH="$PREFIX/bin/"` |
| 6 | `gendbg.sh: Permission denied` | 脚本无执行位 | `chmod +x src/devtools/*.sh` |
| 7 | `tee: /tmp/...` 失败 | Termux 没有 `/tmp` | 日志写到 `$HOME/engine-build.log` |
| 8 | `env: bad interpreter`（depot_tools） | shebang 硬编码 `/usr/bin/env` | `termux-fix-shebang` / `pkg install termux-exec` |

### 3.2 VPC 构建系统

| # | 现象 | 根因 | 解决 |
|---|---|---|---|
| 9 | `unsupported option '-msse' / '-mmmx'` | VPC Makefile 硬编码 x86 选项 | 删除 `src/utils/vpc/Makefile` 中的 `-msse -mmmx` |
| 10 | `sse2neon.h not found` | 未注入 sse2neon include | `CPLUS_INCLUDE_PATH=runtime/android/deps/sse2neon` |
| 11 | `undefined symbol: glob / libiconv_open` | Termux 符号命名与 GNU 不同 | 链接 `-liconv -landroid-glob` |
| 12 | `Bad Syntax in platform_dirs.vpc line:1` | `.vpc` 文件带 UTF-8 BOM | `sed -i '1s/^\xEF\xBB\xBF//'`，批量处理 |
| 13 | `csgo_android_base.mak: No such file` | VPC 丢失 `.mak` 扩展名 | `mv csgo_android_base csgo_android_base.mak` |
| 14 | `No rule to make target .../custom_build_tools/*.sh` | VPC 无法写 `custom_build_tools` 脚本 | 手工 `mkdir` + 空壳脚本 |

### 3.3 依赖补齐

| # | 缺失 | 处理 |
|---|---|---|
| 15 | `expat.h` / `libexpat.a` | 下载 expat 2.5.0，手工交叉编译核心 `.c`（绕过 CMake 4.4 兼容问题） |
| 16 | `libtier0_client.so` / `libvstdlib_client.so` 路径不对 | 从 `game/bin` 复制到 `src/lib/public/androidarm64/release/` |
| 17 | pango/cairo/fontconfig/glib/gobject/pangoft2 | `cp -L $PREFIX/lib/lib*.so runtime/android/install/lib/` |
| 18 | `bzip2_client.a` 等 `.a` 在 `release/` 子目录 | `cp src/lib/common/androidarm64/release/*.a src/lib/common/androidarm64/` |
| 19 | `libcrypto.so` / `libssl.so` | 从 Termux OpenSSL 3 复制，供 `AES_*` deprecated 符号 |

### 3.4 proto / nut 生成（`custom_build_tools`）

这是工作量最大的机械部分。VPC 在 Termux 下无法生成 `*.sh`，每个模块的 proto/nut 依赖都报 `No rule to make target`。

**通用修复模式**：

1. **编 aarch64 的 `protoc` 2.5.0**  
   Termux 的 protobuf 36.2 与 `libprotobuf.a` 2.5.0 ABI 不兼容，必须用项目自带的 2.5.0 源码：
   ```bash
   cd src/thirdparty/protobuf-2.5.0
   CC=clang CXX=clang++ \
   LDFLAGS="-landroid-glob -landroid-support" \
     ./configure --disable-shared --enable-static
   make -C src protoc -j4
```

产物 src/thirdparty/protobuf-2.5.0/src/protoc 是 aarch64，可直接跑。

2. 用这个 protoc 手工生成 .pb.cc / .pb.h
      关键：protoc 2.5.0 要求 -I 和输入文件全绝对路径，混用相对/绝对会报 File does not reside within any path specified。
3. 用 Python 把 .nut 转成 _nut.h：
   ```python
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
4. 为每个 custom_build_tools/*.sh 创建 exit 0 空壳（Debug + Release 各一份）

各模块依赖清单：

模块 proto nut 产物目录
gcsdk steammessages — src/gcsdk/generated_proto/
engine netmessages、network_connection、engine_gcmessages — src/engine/generated_proto/
panorama uifontfile_format — src/panorama/generated_proto_panorama_text_base/
server 7 个 spawn_helper、vscript_server src/game/server/generated_proto/、obj_Server_CSGO_.../
client_panorama 7 个 vscript_client src/game/client_panorama/generated_proto/、src/game/client/obj_Client_Panorama_CSGO_androidarm64_client/release/

关键坑：

· cstrike15/ 子目录的 -I 要排在 game/shared 之前
· Client_Panorama_CSGO 的 makefile 在 src/game/client/，产物路径写 ../client_panorama/generated_proto/，需手动 mkdir
· OBJ_DIR 由 obj_$(NAME)_$(TARGET_PLATFORM)$(TARGET_PLATFORM_EXT)/$(CFG) 拼出，NAME=Client_Panorama_CSGO 大写开头

3.5 V8 5.8 → 6.6.346 迁移（核心难点）

背景：源码头文件是 V8 5.8.283，可用预编译库来自 LayaNative，版本 6.6.346，跨一个大版本。

头文件替换：

```bash
cp -r ~/LayaNative2.0/ThirdParty/v8/include/. src/thirdparty/v8/include/
```

API 变化对照：

5.8 写法 6.6 写法 位置
v8::TryCatch tc; v8::TryCatch tc( v8::Isolate::GetCurrent() ); uijsregistration.h、uiengine.cpp
args.Callee()->GetName() 移除，改用固定字符串 uijsregistration.h:365/405
val->ToNumber() val->ToNumber( ctx ).ToLocalChecked() uiengine.cpp、uievent.cpp
val->ToUint32() val->ToUint32( ctx ).ToLocalChecked() uiengine.cpp:3343
v8::JSON::Parse(str) v8::JSON::Parse( ctx, str ).ToLocalChecked() uiengine.cpp:4182
CreateDefaultPlatform(n) NewDefaultPlatform(n, ...) uiengine.cpp:1525
V8::InitializeICU(path) V8::InitializeICUDefaultLocation(nullptr, path) uiengine.cpp:1535
Isolate::IdleNotification(ms) Isolate::IdleNotificationDeadline(sec) uiengine.cpp:6337/6372
V8::SetFatalErrorHandler(...) 移除，注释 uiengine.cpp:1658
#include "v8-debug.h" 移除（6.6 已删） uiengine.h:49/53
-I 缺 thirdparty/v8/include libplatform.h 找不到 libplatform-export.h makefile INCLUDEDIRS 加

ctx 的取法：

```cpp
v8::Isolate::GetCurrent()->GetCurrentContext()
```

ToLocalChecked() 的风险：非法 JS 值会 abort。原代码的 if (IsEmpty()) 分支永远不会进。若运行时崩，改成 ToLocal( &local ) + 原有错误处理。

3.6 typeinfo for v8::ArrayBuffer::Allocator（V8 RTTI 冲突）

现象：

```
ld.lld: error: undefined symbol: typeinfo for v8::ArrayBuffer::Allocator
>>> referenced by uiengine.o:(typeinfo for panorama::V8ArrayBufferAllocator)
```

根因：

· LayaNative 的 libv8.cr.so 用 -fno-rtti 编译，只导出 vtable，不导出 typeinfo
· uiengine.cpp 里的 panorama::V8ArrayBufferAllocator 继承 v8::ArrayBuffer::Allocator，编译器为子类生成 typeinfo，引用基类 typeinfo
· 直接给 uiengine.cpp 加 -fno-rtti 会连带禁用 tier1/functors.h 的 typeid，报 use of typeid requires -frtti

最终方案：stub .so 补 typeinfo

```cpp
// v8_typeinfo_stub.cpp
#include <typeinfo>
namespace v8 {
class ArrayBuffer {
public:
    class Allocator {
    public:
        virtual ~Allocator() = default;
    };
};
}
extern "C" const std::type_info* __attribute__((used)) __v8_typeinfo_anchor =
    &typeid(v8::ArrayBuffer::Allocator);
```

```bash
"$NDK_BIN/aarch64-linux-android30-clang++" \
  -shared -fPIC -fno-exceptions -frtti \
  -o src/lib/common/androidarm64/libv8_typeinfo_stub.so \
  v8_typeinfo_stub.cpp
```

在 panorama_androidarm64.mak 的 SystemLibraries 末尾加：

```makefile
-L../lib/common/androidarm64 -lv8_typeinfo_stub
```

uiengine.cpp 保持 RTTI 编译，functors.h 的 typeid 不受影响，链接器从 stub 里解析 typeinfo。

关键教训：$(call COMPILE_FILE, $(CFG_CPPFLAGS) -fno-rtti) 无效——COMPILE_FILE 宏展开的是 $(COMPILE_AS_FLAGS)，根本不用 $(1)，参数被完全丢弃。只能用手写编译命令或 target-specific variable。

3.7 图形库与 ABI

# 现象 根因 解决
20 sizeof(long) == 4 静态断言失败 项目自带 glibconfig.h 是 32 位模型（GLIB_SIZEOF_LONG 4） 用 Termux 的 glibconfig.h（GLIB_SIZEOF_LONG 8）覆盖 src/panorama_s1wrapper/thirdparty/pango/glib/glibconfig.h
21 undefined symbol: AFontMatcher_create Android 分支调 libandroid.so 字体 API，未链接 -landroid vguimatsurface 的 SystemLibraries 加 -landroid
22 undefined symbol: AES_set_decrypt_key / AES_decrypt uifontfile.cpp 用 OpenSSL 1.0.1e API，但 libcrypto_client.a 实为 Crypto++，项目 src/external/openssl-1.0.1e/ 只有头文件 复制 Termux libcrypto.so，panorama_text_pango 的 SystemLibraries 加 -lcrypto（Termux OpenSSL 3 保留 deprecated AES_*）

---

4. 关键改动文件清单

文件 改动
scripts/build-android.sh 删 distrobox 块；build_native() 加 CMAKE_SYSROOT；.mak 重命名；LINUX_TOOLS_PATH
android/CMakeLists.txt 变量名 OHOS → Android；__OHOS__ → ANDROID；hilog_ndk.z → log；.so import 路径改 game/bin
android/native/android_main.cpp 1 处 SDL_GetOpenHarmonyInternalStoragePath() → PLATFORM_INTERNAL_STORAGE
src/utils/vpc/Makefile 删除 -msse -mmmx；LINK 加 -liconv -landroid-glob
src/thirdparty/v8/include/ 整体替换为 LayaNative V8 6.6.346 头文件
src/panorama/uiengine.h 注释 v8-debug.h 两行 include
src/panorama/uiengine.cpp 8 处 V8 6.6 API 适配
src/panorama/uievent.cpp 7 处 ToNumber() 加 context
src/public/panorama/uijsregistration.h TryCatch 加 isolate；Callee() 移除
src/panorama_s1wrapper/thirdparty/pango/glib/glibconfig.h 用 Termux aarch64 版本覆盖
src/panorama/panorama_androidarm64.mak INCLUDEDIRS 加 ../thirdparty/v8/include；SystemLibraries 加 -lv8_typeinfo_stub
src/panorama/panorama_text_pango_androidarm64.mak SystemLibraries 加 -lcrypto
src/vguimatsurface/vguimatsurface_androidarm64.mak SystemLibraries 加 -landroid
runtime/android/install/lib/ 新增 libglib-2.0.so、libgobject-2.0.so、libpangoft2-1.0.so、libcrypto.so、libssl.so
src/lib/common/androidarm64/ 新增 libv8*.cr.so（6.6.346）、libv8_typeinfo_stub.so、libcrypto_client.a、libparsifal.a、多个 *_client.a
src/game/{server,client_panorama,client}/ proto .pb.cc、_nut.h、custom_build_tools 空壳脚本

---

5. 遗留问题与风险

5.1 高风险

1. V8 6.6 与源码头文件版本差异
      靠 LayaNative 的 6.6 头文件 + patch 调用点强行对齐。未来升级 V8 必须重新对齐。
2. Termux 的 glib/pango/cairo/OpenSSL 是 API 24 构建
      项目目标是 API 30，链接能过，运行时可能因 API 24 → 30 的符号差异崩溃。发布前应在 x86_64 主机用 NDK r28c 按 API 30 重编。
3. ToLocalChecked() 的运行时 abort 风险
      uiengine.cpp / uievent.cpp 共 11 处改成 ToLocalChecked()。若 JS 传入非法值会 abort。真机跑通后应逐步改成 ToLocal(&local) + 错误处理。
4. libv8_typeinfo_stub.so 的运行时不确定性
      stub 的 typeinfo 和 LayaNative V8 内部 typeinfo 是否同一份，取决于 LayaNative 是否完全禁用 RTTI。CS:GO 对这些 V8 类型基本不用 dynamic_cast，实际风险低，但如果运行时出现 dynamic_cast 相关的未定义行为，需要考虑其他方案。
5. libcrypto.so 的 SONAME 是 libcrypto.so.3
      APK 打包时若只带 libcrypto.so，运行时 linker 找不到 libcrypto.so.3。需在 package_apk 阶段补一份 libcrypto.so.3。

5.2 中风险

6. panorama 的 V8 调用点可能还有遗漏
      uiengine.cpp 有 6000+ 行，后续编译可能还有 MaybeLocal 转换、Context::New 签名等。
7. custom_build_tools 空壳脚本只解决"脚本存在"
      .pb.cc / _nut.h 靠手工生成，脚本本身是 exit 0。若 VPC 重新生成 makefile，空壳和产物都要重做。
8. android-build-tools.py 的 stage() 硬编码 linux-x86_64
      在 Termux 上找不到 llvm-readelf 和 libc++_shared.so，需改成动态 host tag 检测。

5.3 低风险

9. .a 路径不一致
      src/lib/common/androidarm64/release/ 和顶层各有一份，需在 foundation / engine 后加一步 rsync。

---

6. 可复现脚本

6.1 VPC 空壳脚本生成

```bash
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

6.2 proto 生成（全绝对路径）

```bash
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

6.3 nut → h

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

7. 后续建议

7.1 短期（让 engine 完整跑通）

1. 继续跑 all-targets，修复 Client_Panorama_CSGO / Server_CSGO 剩余 proto/nut/.a 缺失
2. 修 android-build-tools.py 的 linux-x86_64 硬编码
3. 补 libcrypto.so.3 到 APK 打包清单
4. 跑 native + package + verify，打出第一个完整 APK

7.2 中期（提升可维护性）

5. 把所有手工 patch 整理成 android/patches/termux-*.patch，在 build-android.sh 里自动应用
6. 把 proto/nut/空壳脚本的生成逻辑写进 scripts/android-build-tools.py
7. 把 V8 6.6 的 API 适配封装成 v8_compat.h 的 inline 辅助函数

7.3 长期（生产级构建）

8. V8 换回源码编译：在 x86_64 主机按 DEPS 的 commit 编 V8 5.8.283，彻底消除版本差异
9. 图形库按 API 30 交叉编译：glib/pango/cairo/fontconfig 用 NDK r28c 重编
10. CI 矩阵：x86_64 构建重型依赖 → Termux 只做 native / package / install

---

8. 一句话总结

这次移植的每一步都在和三种"假设"搏斗：路径假设（/bin/bash、/tmp、linux-x86_64）、版本假设（V8 5.8 vs 6.6、protobuf 2.5 vs 36、glib 32 vs 64）、产物假设（.mak 扩展名、custom_build_tools 脚本、.so/.a 路径）。
逐类击破后，最难的一关——V8 6.6 的 RTTI typeinfo 冲突——用一个 2KB 的 stub .so 绕过，panorama 链接成功，engine 主流水线可以继续推进到 Client_Panorama_CSGO 和 Server_CSGO。