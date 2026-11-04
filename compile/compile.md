# linux 编译

你是为了学习源码，建议先编译 Debug 版本，方便下断点、查看变量。使用已安装的 GCC 13。

在终端执行：

cd /home/leo/dev/RocksDB/rocksdb

cmake -S . -B build-debug \
  -DCMAKE_C_COMPILER=gcc-13 \
  -DCMAKE_CXX_COMPILER=g++-13 \
  -DCMAKE_BUILD_TYPE=Debug \
  -DWITH_GFLAGS=OFF \
  -DWITH_TESTS=ON \
  -DWITH_BENCHMARK_TOOLS=OFF \
  -DWITH_TOOLS=OFF \
  -DWITH_CORE_TOOLS=OFF

然后编译库和一个基础测试：

cmake --build build-debug --target rocksdb db_basic_test -j4

运行测试，确认基本读写正常：

./build-debug/db_basic_test

想跟踪执行过程，可以用 GDB：

gdb --args ./build-debug/db_basic_test

进入 GDB 后：

break rocksdb::DBImpl::Write
run

构建产物都在 build-debug/ 中。先用 -j4 控制内存占用；内存充足时可以提高并行数。以上是编译步骤，我还没有实际执行构建。

# mac 编译

与上面的 Linux 流程相同，使用 CMake 构建 Debug 库和 `db_basic_test`，关闭 gflags、基准工具和命令行工具。macOS 使用系统自带的 Apple Clang 编译，并使用 LLDB 调试。

以下命令适用于当前这台 Apple Silicon Mac，源码目录是 `/Users/leo/doc/RocksDB/rocksdb`。

## 准备 CMake

当前电脑已经安装 Xcode Command Line Tools，可以使用 `/usr/bin/clang`、`/usr/bin/clang++` 和 `lldb`，但还没有检测到 CMake 或 Homebrew。

可以从 [CMake 官方下载页](https://cmake.org/download/) 下载稳定版 macOS universal 的 `.dmg`，将 `CMake.app` 放到 `/Applications` 后，在当前终端执行：

```bash
export PATH="/Applications/CMake.app/Contents/bin:$PATH"
cmake --version
```

如果已经安装 Homebrew，也可以用 `brew install cmake` 安装 CMake。这个构建不需要额外安装 GCC、Ninja、gflags 或压缩库；测试框架随源码提供。

## 配置 Debug 构建

当前环境的 `SDKROOT` 指向 `MacOSX27.0.sdk`，之前的编译探测发现它会导致链接器报 `unknown architecture arm64e.x1`。本机的 `MacOSX26.5.sdk` 已通过 C++20 编译、链接和运行探测，因此显式选择它，同时通过 `CMAKE_OSX_SYSROOT` 固定 CMake 使用的 SDK。

```bash
cd /Users/leo/doc/RocksDB/rocksdb

export SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX26.5.sdk

cmake -S . -B build-debug-macos -G "Unix Makefiles" \
  -DCMAKE_C_COMPILER=/usr/bin/clang \
  -DCMAKE_CXX_COMPILER=/usr/bin/clang++ \
  -DCMAKE_OSX_ARCHITECTURES=arm64 \
  -DCMAKE_OSX_SYSROOT="$SDKROOT" \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_C_FLAGS_DEBUG="-O0 -g" \
  -DCMAKE_CXX_FLAGS_DEBUG="-O0 -g" \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
  -DWITH_GFLAGS=OFF \
  -DWITH_TESTS=ON \
  -DWITH_BENCHMARK_TOOLS=OFF \
  -DWITH_TOOLS=OFF \
  -DWITH_CORE_TOOLS=OFF
```

`-O0 -g` 用于关闭优化、保留调试信息，方便单步和查看变量。`compile_commands.json` 用于编辑器的源码跳转。单独使用 `build-debug-macos/`，避免复用 Linux 或其他编译器生成的 CMake 缓存。

## 编译和运行

先保持与 Linux 流程相同的 `-j4`，控制内存占用：

```bash
cmake --build build-debug-macos --target rocksdb db_basic_test -j4
```

学习时可以先只运行一个包含写入、Flush 和读取的用例：

```bash
./build-debug-macos/db_basic_test --gtest_filter=DBBasicTest.Flush
```

如果要运行与 Linux 文档相同的整个基础测试程序：

```bash
./build-debug-macos/db_basic_test
```

## 使用 LLDB 跟踪调用链

```bash
lldb -- ./build-debug-macos/db_basic_test --gtest_filter=DBBasicTest.Flush
```

进入 LLDB 后执行：

```text
breakpoint set -n 'rocksdb::DBImpl::Write'
run
```

停在断点后，使用 `bt` 查看调用栈、`step` 进入函数、`next` 执行下一行、`frame variable` 查看局部变量、`continue` 继续运行。命令说明可参考 [LLDB 官方教程](https://lldb.llvm.org/use/tutorial.html)。

构建产物位于 `build-debug-macos/`。以上步骤已对照当前仓库的 CMake 选项检查，并通过独立的小程序验证了 Clang、C++20、ARM64、26.5 SDK 和 RocksDB 公共头文件的组合；由于当前尚未安装 CMake，还没有实际执行 CMake 配置、完整构建或测试。

# mac 编译 db_bench
```zsh
export SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX26.5.sdk
export CPATH="$(brew --prefix)/include${CPATH:+:$CPATH}"
export LIBRARY_PATH="$(brew --prefix)/lib${LIBRARY_PATH:+:$LIBRARY_PATH}"

AUTO_CLEAN=1 DEBUG_LEVEL=0 make -j"$(sysctl -n hw.ncpu)" db_bench
```

然后检查：
```zsh
grep -E -- '-DSNAPPY|-DGFLAGS' make_config.mk
```
如果有输出，那么说明完成了编译

