## 我应该怎么学习 rocksdb 源码？

建议按 “跑通测试 → 跟踪一次写入 → 跟踪一次读取 → 理解后台整理” 的顺序学习。先建立数据流，再深入细节，比逐个目录阅读更容易理解。

你当前这份代码要求 C++20，学习时用已安装的 GCC/G++ 13 编译即可。C++ 方面先掌握 RAII、智能指针、移动语义、模板基础、互斥锁和条件变量；遇到高级用法再补，不必先学完所有 C++20 特性。

你当前仓库已经有很好的导读，可以直接配合源码阅读：

- [写入流程导读](../rocksdb/docs/components/write_flow/index.md)
- [读取流程导读](../rocksdb/docs/components/read_flow/index.md)

### 1. 先理解整体数据流

先记住这两条路径：

写入：应用 → WAL 日志 + MemTable → Flush → SST 文件
读取：应用 → MemTable → Immutable MemTable → SST 文件

后台：Compaction 合并 SST，清理过期版本和删除标记

带着三个问题阅读：数据存在哪里？如何找到它？崩溃后如何恢复？

### 2. 从公开接口和小程序开始

先看：

- include/rocksdb/db.h (rocksdb/include/rocksdb/db.h)：Open、Put、Get、Delete、Write、迭代器。
- examples/simple_example.cc (rocksdb/examples/simple_example.cc)：基本使用方式。
- include/rocksdb/options.h (rocksdb/include/rocksdb/options.h)：遇到参数时查阅，暂时不用通读。

自己写一个小实验：写入同一个 key 两次、读取、删除、关闭数据库、重新打开再读取。之后增加快照，观察更新前后的可见性。

### 3. 跟踪一次写入

重点看 db/db_impl/db_impl_write.cc (rocksdb/db/db_impl/db_impl_write.cc)，从 DBImpl::Write 开始追踪。

第一遍只回答：

- WriteBatch 如何表达一批更新？
- WAL 在哪里写入？
- 数据什么时候进入 MemTable？
- Sequence Number 如何分配？
- 写入返回时，数据持久化到了什么程度？

这里会遇到写入分组、并发和流水线等分支。先沿着普通写入路径走通，之后再研究优化。

### 4. 跟踪一次读取

从 db/db_impl/db_impl.cc (rocksdb/db/db_impl/db_impl.cc) 中的 DBImpl::Get 相关实现开始，逐步看：

- 如何按顺序查找内存和 SST？
- 如何判断一个版本对当前读取可见？
- Bloom Filter、索引和 Block Cache 分别减少了什么开销？
- 删除标记为什么能让旧值“消失”？

这一步再看 db/version_set.cc (rocksdb/db/version_set.cc) 和 table/block_based/，会更有目的。

### 5. 再研究 Flush 和 Compaction

重点文件：

- db/flush_job.cc (rocksdb/db/flush_job.cc)：MemTable 如何生成 SST。
- db/compaction/compaction_job.cc (rocksdb/db/compaction/compaction_job.cc)：如何执行合并。
- db/version_set.cc (rocksdb/db/version_set.cc)：文件集合如何更新，元数据如何管理。

重点理解：为什么要分层、为什么会产生写放大、旧版本何时可以删除，以及快照为什么会影响清理。

### 6. 用测试验证理解

阅读某个功能时，在附近找对应的 *_test.cc。选择一个小测试：

1. 先预测结果。
2. 运行或调试，核对预测。
3. 修改一个条件，例如增加一次更新、删除或快照。
4. 解释结果为什么变化。

每读完一条路径，画一张调用关系图，并写下“输入是什么、改变了什么状态、失败后怎么办”。这比记录大量函数名更有用。

你的第一个学习目标可以很具体：能够解释 Put("k", "v") 返回前发生了什么，以及紧接着的 Get("k") 为什么能读到它。 完成这条闭环后，再扩展到 SST、Compaction 和恢复流程。
