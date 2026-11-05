# linux 编译

你是为了学习源码，建议先编译 Debug 版本，方便下断点、查看变量。使用已安装的 GCC 13。

在终端执行：

cd /home/leo/dev/RocksDB

cmake -S rocksdb -B build-debug \
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

构建产物都在 build-debug/ 中，与 rocksdb/ 源码目录并列。`-S` 指定源码目录，`-B` 指定构建目录，相对路径以执行命令时的工作目录为基准。先用 -j4 控制内存占用；内存充足时可以提高并行数。以上是 Linux 编译步骤，尚未在 Linux 上实际执行构建。

# mac 编译

与上面的 Linux 流程相同，使用 CMake 构建 Debug 库和 `db_basic_test`，关闭 gflags、基准工具和命令行工具。macOS 使用系统自带的 Apple Clang 编译，并使用 LLDB 调试。

以下命令适用于当前这台 Apple Silicon Mac。在 `/Users/leo/doc/RocksDB` 中执行，源码目录和构建目录并列：

```text
RocksDB/
├── rocksdb/               # 源码
├── build-debug-macos/     # macOS Debug 构建产物
└── rocksdb_notes/         # 学习笔记
```

## 准备 CMake

需要安装 Xcode Command Line Tools 和 CMake。当前电脑已安装这些工具，可以使用 `/usr/bin/clang`、`/usr/bin/clang++`、`lldb` 和 `cmake`。如果在新电脑上配置，可以按下面的方法安装 CMake。

可以从 [CMake 官方下载页](https://cmake.org/download/) 下载稳定版 macOS universal 的 `.dmg`，将 `CMake.app` 放到 `/Applications` 后，在当前终端执行：

```bash
export PATH="/Applications/CMake.app/Contents/bin:$PATH"
cmake --version
```

如果已经安装 Homebrew，也可以用 `brew install cmake` 安装 CMake。这个构建不需要额外安装 GCC、Ninja、gflags 或压缩库；测试框架随源码提供。

## 配置 Debug 构建

当前环境的 `SDKROOT` 指向 `MacOSX27.0.sdk`，之前的编译探测发现它会导致链接器报 `unknown architecture arm64e.x1`。本机的 `MacOSX26.5.sdk` 已通过 C++20 编译、链接和运行探测，因此显式选择它，同时通过 `CMAKE_OSX_SYSROOT` 固定 CMake 使用的 SDK。

```bash
cd /Users/leo/doc/RocksDB

export SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX26.5.sdk

cmake -S rocksdb -B build-debug-macos -G "Unix Makefiles" \
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

`-O0 -g` 用于关闭优化、保留调试信息，方便单步和查看变量。`compile_commands.json` 用于编辑器的源码跳转。`-B build-debug-macos` 将产物放在当前目录下，与 `-S rocksdb` 指定的源码目录并列，并避免复用 Linux 或其他编译器生成的 CMake 缓存。

CMake 缓存和生成文件包含绝对路径。已有构建目录移动后，需要用 `cmake --fresh` 加上上述配置参数重新生成缓存，再重新编译；只移动目录不能完成迁移。

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

构建产物位于 `/Users/leo/doc/RocksDB/build-debug-macos/`，与源码目录并列。以上配置使用 Clang、C++20、ARM64 和 26.5 SDK，已在该目录完成 `rocksdb`、`db_basic_test` 构建，并通过 `DBBasicTest.Flush` 和 `DBBasicTest.GetNewerVersionPresent` 两个用例。

# 编译 db_bench

## mac 编译 db_bench
```zsh
cd /Users/leo/doc/RocksDB/rocksdb

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

## linux 编译 db_bench

你当前已有 build/，且已启用 db_bench，直接编译：

```bash
cd /home/leo/dev/RocksDB
cmake --build build --target db_bench -j8
./build/db_bench --help
```

当前 build/ 配置是 Debug。 如果要测性能，建议单独建 Release 目录：

```bash
cmake -S rocksdb -B build-release \
  -DCMAKE_BUILD_TYPE=Release \
  -DWITH_BENCHMARK_TOOLS=ON \
  -DWITH_GFLAGS=ON \
  -DWITH_TESTS=OFF

cmake --build build-release --target db_bench -j8
./build-release/db_bench --help
```

也可以在源码目录用 Make：

```bash
cd /home/leo/dev/RocksDB/rocksdb
make -j8 db_bench DEBUG_LEVEL=0
./db_bench --help
```

db_bench 需要 gflags；若提示缺失，Ubuntu/Debian 下安装 sudo apt install libgflags-dev。内存不足时将 -j8 调小。
