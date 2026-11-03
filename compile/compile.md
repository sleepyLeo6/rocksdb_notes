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