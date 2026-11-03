# 跟踪一次 Put

建议先跟踪一次 Put 从接口进入 WAL、再进入 MemTable，暂时跳过后台 Flush 和 Compaction。

你这份源码的主线是：

DBImpl::Put
  → DB::Put                  把 key/value 封装成 WriteBatch
  → DBImpl::Write
  → DBImpl::WriteImpl         写线程协调、分组、分配序列号
      → WriteToWAL           写 WAL（默认启用）
          → log::Writer::AddRecord
      → WriteBatchInternal::InsertInto
          → MemTable::Add    写入内存表

1. 用单个测试启动 GDB

在 rocksdb 源码目录执行：

```bash
gdb --args build-debug/db_basic_test --gtest_filter=DBBasicTest.GetNewerVersionPresent
```

这个测试可以作为观察写入的入口。不要直接运行整个测试套件，否则断点会命中太多次。

2. 先只设置三个断点

```bash
set breakpoint pending on
b rocksdb::DBImpl::WriteImpl
b rocksdb::DBImpl::WriteToWAL
b rocksdb::MemTable::Add
run
```

共享库还没加载时，GDB 可能提示断点暂时 pending，这是正常的。

实际上这里最后会追踪到 `/home/leo/dev/RocksDB/rocksdb/build-debug/librocksdb.so.11`<br>
还需要再加一个断点：`break /home/leo/dev/RocksDB/rocksdb/db/db_impl/db_impl_write.cc:907`<br>
然后再 `continue`

每次停下后：

bt          # 查看谁调用了当前函数
info args   # 查看参数
n           # 执行下一行，不进入函数
s           # 进入函数
finish      # 执行到当前函数返回
c           # 继续到下一个断点

用 c 跳到关键节点，再用 n 看附近逻辑，比从入口一路 s 更容易理解。

3. 对照这几个源码位置阅读

| 位置 | 重点 |
|---|---|
| DB::Put (rocksdb/db/db_impl/db_impl_write.cc:3650) | 单条写入如何变成 WriteBatch |
| DBImpl::Write (rocksdb/db/db_impl/db_impl_write.cc:230) | 如何进入写入主流程 |
| DBImpl::WriteImpl (rocksdb/db/db_impl/db_impl_write.cc:891) | 写分组、序列号、WAL 和 MemTable 的调用顺序 |
| DBImpl::WriteToWAL (rocksdb/db/db_impl/db_impl_write.cc:2330) | WAL 写入 |
| MemTable::Add (rocksdb/db/memtable.cc:1132) | 序列号、记录类型和 key/value 如何编码并插入 |

在 WriteImpl 中重点关注 write_options.disableWAL、write_options.sync，在 MemTable::Add 中关注 s、type、key、value。

注意：默认 Put 返回时，数据通常已经进入 WAL 和 MemTable，但还没有生成 SST；写入 WAL 也不等于完成磁盘同步，后者受 sync 等选项影响。 等这条主线走通，再单独跟踪 Flush → SST 会清楚很多。

# DBImpl::WriteImpl 解析

## 回答 1

WriteImpl() 是 RocksDB 写入流程的核心调度函数：组织并发写请求、写 WAL、写 MemTable，最后发布写入结果。

实现从 db_impl_write.cc:891 (rocksdb/db/db_impl/db_impl_write.cc:891) 开始。这个函数分支很多，先沿着普通 Put() 的路径理解最容易。

### 1. 检查参数和写入配置

首先检查 batch 和选项是否合法，例如：

if (my_batch == nullptr) {
  return Status::InvalidArgument("Batch is nullptr!");
}

还会检查时间戳是否设置、保护信息字节数是否合法、某些选项能否同时使用等。

如果启用了特殊写入模式，则转到相应流程，例如：

if (immutable_db_options_.enable_pipelined_write) {
  return finish_write(PipelinedWriteImpl(...));
}

下面解释的是普通写入路径。

### 2. 把当前请求包装成 Writer，进入写入队列

WriteThread::Writer w(write_options, my_batch, ...);
write_thread_.JoinBatchGroup(&w);

这里三个对象的职责不同：

| 对象 | 含义 |
|---|---|
| my_batch | 当前请求要写入的记录 |
| w | 当前写请求，包括 batch、选项、状态和序列号等 |
| write_group | 一组可以一起处理的写请求 |

多个线程可能同时调用 Put()。RocksDB 会组织这些请求，选出一个 leader（组长），协调这一组写入。

所以，调用 WriteImpl() 的线程不一定亲自完成全部工作：它的 WAL 写入可能由另一个请求的 leader 完成。

### 3. 根据当前线程的角色执行

JoinBatchGroup() 返回后，函数检查 w.state：

- STATE_COMPLETED：请求已由其他线程协助完成，可以返回。
- STATE_PARALLEL_MEMTABLE_WRITER：当前线程负责参与并行写 MemTable。
- STATE_GROUP_LEADER：当前线程成为 leader，继续组织整组写入。

这也解释了为什么函数前半部分就出现了 InsertInto() 和 return：它们处理的是不同线程角色。

### 4. Leader 做写入准备，组织写入组

status = PreprocessWrite(write_options, &wal_context, &write_context);

write_thread_.EnterAsBatchGroupLeader(&w, &write_group);

准备工作包括处理写入压力、必要的 WAL/MemTable 切换等。之后，leader 将符合条件的排队请求组成一个写入组。

例如：

线程 A：batch [Put(k1, v1)]  ← leader
线程 B：batch [Put(k2, v2)]
线程 C：batch [Delete(k3)]
             ↓
         一个 write_group

这些仍是各自独立的写请求，只是在内部一起处理以提高效率。

### 5. 为记录安排序列号，写入 WAL

普通路径会根据记录数量计算需要使用的序列号范围：

Put(k1, v1) → sequence 101
Put(k2, v2) → sequence 102
Delete(k3) → sequence 103

序列号用于确定数据版本和快照可见性。

如果没有禁用 WAL，则调用：

io_s = WriteGroupToWAL(...);

WAL 用于故障恢复。是否执行持久化同步，还取决于 sync 等选项。

这里的基本顺序是：

先写 WAL → 再写 MemTable

这样即使 MemTable 中的数据尚未 flush 成 SST，也可以借助 WAL 恢复。

### 6. 将记录写入 MemTable

串行模式下，由 leader 处理整组：

w.status = WriteBatchInternal::InsertInto(
    write_group, current_sequence, ...);

并行模式下，先唤起参与者：

write_thread_.LaunchParallelMemTableWriters(&write_group);

各线程再通过 InsertInto() 写入各自负责的记录。

InsertInto() 是继续追踪“key/value 如何进入 MemTable”的关键入口。 它会解析 batch，将记录应用到对应列族的 MemTable。

### 7. 发布序列号，通知其他请求完成

成功完成相应写入后：

versions_->SetLastSequence(last_sequence);

这一步发布已完成写入的序列号，让后续读取按相应序列号看到新版本。代码特别避免发布只完成了一部分的 batch。

接着退出写入组：

write_thread_.ExitAsBatchGroupLeader(write_group, status);

协调其他等待线程完成，最终返回当前请求的状态。

整体主线是：

检查参数
   ↓
请求进入队列
   ↓
Leader 组织写入组
   ↓
准备写入、安排序列号
   ↓
写 WAL
   ↓
写 MemTable
   ↓
发布完成的序列号
   ↓
通知等待线程、返回 Status

你前面关注的 WriteBatch，在这里成为实际处理的输入；而 WriteImpl() 主要负责协调这些输入以正确的顺序进入 WAL 和 MemTable。

## 回答 2

这个函数从 db_impl_write.cc:891 (rocksdb/db/db_impl/db_impl_write.cc:891) 开始。它的核心职责是：把一个写请求放进并发写入体系，协调 WAL、MemTable 和序列号，最后返回这个请求的执行结果。

它比较长，是因为普通 Put()、事务写入、流水线写入、无序写入以及 Blob 写入等都经过这里。阅读时可以先抓住普通 Put() 的路径，再理解特殊分支。

下面按源码执行顺序解释。

1. 参数分别表示什么

```cpp
Status DBImpl::WriteImpl(
    const WriteOptions& write_options,
    WriteBatch* my_batch,
    WriteCallback* callback,
    UserWriteCallback* user_write_cb,
    uint64_t* wal_used,
    uint64_t log_ref,
    bool disable_memtable,
    uint64_t* seq_used,
    size_t batch_cnt,
    PreReleaseCallback* pre_release_callback,
    PostMemTableCallback* post_memtable_callback,
    std::shared_ptr<WriteBatchWithIndex> wbwi,
    WriteBatch* trace_batch_override,
    bool skip_blob_direct_write_transform,
    const DBImpl::DeferredPutEntityBatch* deferred_put_entities)
```

可以分为几组理解：

| 参数 | 用途 |
|---|---|
| write_options | 控制 WAL、同步、低优先级、保护信息等 |
| my_batch | 当前请求要执行的操作集合 |
| callback、user_write_cb | 可选的写入回调 |
| wal_used | 输出本次请求使用的 WAL 编号 |
| log_ref | 内部使用的日志引用信息，主要涉及事务等路径 |
| disable_memtable | 是否跳过 MemTable 写入，支持只写 WAL 的操作 |
| seq_used | 输出本次请求使用的起始序列号 |
| batch_cnt | 特殊序列号分配模式下使用的批次数量 |
| pre_release_callback | 在 WAL 阶段后、MemTable 阶段前调用 |
| post_memtable_callback | MemTable 阶段完成后的回调 |
| wbwi | 支持将 WriteBatchWithIndex 作为 MemTable 导入 |
| trace_batch_override | 指定追踪记录使用的逻辑 batch |
| 最后两个参数 | 控制 Blob 直接写入转换以及延后处理的实体写入 |

对于你前面的普通 Put()，主要关注 write_options 和 my_batch 即可，其余参数大多使用默认值。

2. 检查输入是否满足写入要求

函数开头先用断言检查内部约定：

assert(!seq_per_batch_ || batch_cnt != 0);

启用了“按 batch 分配序列号”的模式，就必须提供批次数量。

另一个断言检查 batch 的保护信息配置与写入选项是否一致。这也衔接了前面 UpdateProtectionInfo() 的处理。

随后进行运行时检查，例如：

if (my_batch == nullptr) {
  return Status::InvalidArgument("Batch is nullptr!");
}

其他检查包括：

- 要写入 MemTable 的记录是否已经设置必要的用户时间戳。
- 限速优先级是否合法。
- protection_bytes_per_key 是否为 0 或 8。
- 禁用 WAL 是否与其他配置冲突。

这里要区分：assert 检查内部程序约定；返回 Status 的检查向调用者报告错误。

3. 处理 tracing 和不兼容的配置

WriteBatch* trace_batch =
    trace_batch_override != nullptr ? trace_batch_override : my_batch;

实际写入的 batch 可能因内部转换而改变，因此追踪时可以另外指定“逻辑上的原始操作”。

如果 tracing 不要求保持实际写入顺序，就可以在进入写入队列前记录；要求保持顺序的 tracing 会在后面组队完成后记录。

然后检查配置组合。例如：

if (write_options.sync && write_options.disableWAL) {
  return Status::InvalidArgument("Sync writes has to enable WAL.");
}

sync 要求同步 WAL，而 disableWAL 禁用 WAL，这两个选项不能同时生效。

此外，流水线写入与某些事务模式、无序写入、部分回调之间也有兼容性限制，函数会在这里提前拒绝。

4. 处理 WBWI 导入和低优先级写入

[WBWI 是什么](../wiki/WBWI.md)

如果传入 wbwi，函数会区分：

- 事务提交时导入。
- 直接调用 IngestWriteBatchWithIndex() 导入。

这条路径有专门限制，例如不能与无序写入、流水线写入同时使用。

对于低优先级请求：

```cpp
if (write_options.low_pri) {
  Status s = ThrottleLowPriWritesIfNeeded(write_options, my_batch);
  if (!s.ok()) {
    return s;
  }
}
```

它可能在这里被限速，失败则提前返回。

普通 Put() 如果没有这些配置，可以继续往下看。

5. 提前分流到特殊写入路径

有几个分支会直接转到其他函数，不再走后面的普通路径。

第一种是双写队列中的 WAL-only 请求：[位置](../../rocksdb/db/db_impl/db_impl_write.cc#L1037)

```cpp
if (two_write_queues_ && disable_memtable) {
  return WriteImplWALOnly(...);
}
```

它使用专门的 nonmem_write_thread_，服务于事务等只需要记录 WAL 的阶段。

第二种是[无序写入](../wiki/unorderd_write.md)：[位置](../../rocksdb/db/db_impl/db_impl_write.cc#L1083)

```cpp
if (immutable_db_options_.unordered_write) {
  // 先执行 WAL 阶段
  WriteImplWALOnly(...);

  // 再执行 MemTable 阶段
  UnorderedWriteMemtable(...);
}
```

这种模式采用不同的排序和可见性安排，不能直接套用后面普通路径的解释。

第三种是[流水线写入](../wiki/pipelined_write.md)：[位置](../../rocksdb/db/db_impl/db_impl_write.cc#L1111)

```cpp
if (immutable_db_options_.enable_pipelined_write) {
  return finish_write(PipelinedWriteImpl(...));
}
```

它将 WAL 和 MemTable 阶段组织成流水线，让不同写入组的阶段可以重叠执行。

此外，这附近还创建了 Blob 写入转换及失败回滚所需的对象。finish_write() 是一个局部 lambda：成功时解除 Blob 回滚保护，再返回状态。没有启用相关功能时，可以把它理解成返回状态的收尾包装。

6. 普通路径：创建 Writer 并进入队列 [位置](../../rocksdb/db/db_impl/db_impl_write.cc#L1118)

从大约第 1118 行开始：
```cpp
WriteThread::Writer w(write_options, my_batch, ...);
write_thread_.JoinBatchGroup(&w);
```

Writer 是对一个写请求的包装，除了 batch，还保存：
- 写入选项。
- 当前执行状态。
- 分配的序列号。
- WAL 编号。
- 回调和执行结果。

注意三个层次：<br>
WriteBatch：一组操作记录<br>
Writer：一次写请求<br>
WriteGroup：一组一起处理的写请求<br>

例如三个线程同时调用 Put()：
```
线程 A → Writer A → batch [Put(k1, v1)]
线程 B → Writer B → batch [Put(k2, v2)]
线程 C → Writer C → batch [Put(k3, v3)]
```

它们可能被组织成一个 WriteGroup，由其中一个线程作为 leader 协调处理。是否能组合，还取决于选项是否兼容等条件。

7. JoinBatchGroup 返回后，先判断当前线程的角色

这里很容易读迷糊：源码在 leader 的主流程之前，就出现了写 MemTable 的代码。

原因是 JoinBatchGroup() 包含排队等待和协调过程。它返回时，当前请求可能处于不同状态。

如果是：

STATE_PARALLEL_MEMTABLE_WRITER

说明当前线程是参与并行写 MemTable 的成员，执行：

WriteBatchInternal::InsertInto(&w, w.sequence, ...);

完成后，通过：

write_thread_.CompleteParallelMemTableWriter(&w)

参与协调组的结束。负责收尾的线程会发布完成序列号并通知组内成员。

如果是：

STATE_COMPLETED

说明请求已经完成，可以填写输出参数并返回：

return finish_write(w.FinalStatus());

只有剩下的：

STATE_GROUP_LEADER

才会继续走下面的 leader 主流程。

因此，一次调用不会从头到尾执行所有分支；它会根据线程角色走不同路径。

8. Leader 执行写入前的准备

Leader 创建几个上下文：

WriteContext write_context;
WalContext wal_context(write_options.sync);
WriteThread::WriteGroup write_group;

随后调用：

status = PreprocessWrite(write_options, &wal_context, &write_context);

这个函数会处理多种写入前条件，包括：

- 数据库是否已因后台错误停止写入。
- WAL 总大小是否超过限制。
- WriteBufferManager 是否要求 flush。
- 是否有待处理的 MemTable flush。
- 是否需要延迟或暂停写入。
- WAL 同步相关的协调。

所以调用 Put() 时出现等待，不一定是在写磁盘，也可能停在这里等待内存压力或后台积压缓解。

之后：

write_thread_.EnterAsBatchGroupLeader(&w, &write_group);

正式组织本次处理的写请求组。

即使前面的准备失败，仍需要正确处理组内请求的完成与唤醒，因此不能简单地在任何位置直接返回。

9. 必要时转换 Blob 写入内容

当前源码包含 Blob 直接写入逻辑。

普通路径把转换放在 PreprocessWrite() 之后，是因为准备阶段可能切换 MemTable；转换后的 Blob 索引必须与目标 MemTable 对应的 Blob 文件代次匹配。

这一阶段可能：

- 将 value 写入 Blob 文件。
- 将 batch 中相应内容转换为 Blob 索引。
- 按写入选项同步相关 Blob 文件。
- 在失败时通过前面建立的保护对象尝试回滚。

如果没有启用 Blob 直接写入，可以跳过这一块阅读。

10. 检查回调、统计记录数，决定能否并行写 MemTable

初始条件是：

bool parallel =
    immutable_db_options_.allow_concurrent_memtable_write &&
    write_group.size > 1;

然后遍历写入组：

for (auto* writer : write_group) {
  if (writer->CheckCallback(this)) {
    ...
  }
}

这里会统计有效记录数量、字节数、批次数量，同时进一步检查并行条件：

parallel = parallel && !writer->batch->HasMerge();

当前实现中，组内存在有效的 Merge batch，就不能走这里的并行 MemTable 写入路径。

回调失败的请求会被标记，后面的处理会跳过相应请求。

11. 计算序列号范围

seq_inc = seq_per_batch_ ? valid_batches : total_count;

普通模式通常按记录数量消耗序列号；特殊模式按 batch 数量处理。

例如当前已完成序列号为 100，本组有三条普通记录，则本组使用：

101、102、103

代码中：

current_sequence = last_sequence + 1;
last_sequence += seq_inc;

这里要区分：

- 分配序列号：决定记录使用哪个版本编号。
- 发布序列号：向读取流程宣布哪些编号的写入已经完成。

分配后并不意味着数据已经对普通读取可见。

12. 写入 WAL

普通单队列路径调用：

io_s = WriteGroupToWAL(
    write_group, ...,
    last_sequence + 1, ...);

双队列路径则调用：

ConcurrentWriteGroupToWAL(...);

如果 disableWAL=true，跳过 WAL，并标记数据库存在尚未持久化的数据：

has_unpersisted_data_.store(true, ...);

启用 WAL 时，这一阶段为故障恢复记录写入内容。需要同步时，还会处理 WAL 同步状态，必要时更新 MANIFEST 中的相关信息。

普通路径中，WAL 阶段成功后，才继续进入 MemTable 阶段。

13. 为各个 Writer 设置起始序列号，执行前置回调

组内每个请求需要知道自己的起始序列号：

writer->sequence = next_sequence;

例如：

Writer A 有 2 条记录 → 起始序列号 101
Writer B 有 1 条记录 → 起始序列号 103

接着执行：

writer->pre_release_callback->Callback(...);

源码明确规定这个回调位于 WAL 写入之后、MemTable 写入之前。普通 Put() 没有传入该回调，会跳过。

14. 写入 MemTable

这是记录实际进入内存数据结构的阶段。

串行模式：

WriteBatchInternal::InsertInto(
    write_group, current_sequence, ...);

Leader 将整个写入组的记录应用到对应列族的 MemTable。

并行模式：

write_thread_.LaunchParallelMemTableWriters(&write_group);

通知其他线程参与写入，然后 leader 自己也执行：

WriteBatchInternal::InsertInto(&w, w.sequence, ...);

前面第 7 步看到的 follower 分支，就是被这里唤起后执行的工作。

这段代码跨越了函数的前后两部分：

Leader：LaunchParallelMemTableWriters()
                   ↓
Follower：前半部分的 InsertInto()
Leader：后半部分的 InsertInto()
                   ↓
         协调所有参与者完成

15. 检查错误，处理特殊导入

随后分别处理 WAL I/O 错误、回调错误和 MemTable 插入错误。

这些错误的处理不同，因为它们发生的阶段不同。例如 WAL 已经成功，而 MemTable 插入失败时，不能当作完全没有发生过写入，需要调用：

HandleMemTableInsertFailure(...)

处理数据库状态。

如果本次请求带有 wbwi，还会在这里调用：

IngestWBWIAsMemtable(...)

将 WBWI 对应的数据作为 MemTable 导入，并为其安排相应序列号范围。

16. 完成回调、发布序列号、结束写入组

负责结束写入组的线程执行 post_memtable_callback，然后在相应写入成功的条件下：

versions_->SetLastSequence(last_sequence);

发布这个组已完成的序列号。

普通读取通常基于已发布的序列号确定可见版本，因此这个步骤也是普通路径中保证 batch 可见性的重要环节。不能只因为某条记录已插入 MemTable，就认定它已经对读取可见。

最后：

write_thread_.ExitAsBatchGroupLeader(write_group, status);

结束写入组并协调等待线程完成，结合当前 Writer 的最终结果返回：

status = w.FinalStatus();
return finish_write(status);

对于你正在调试的普通 Put()，可以先用这条主线阅读：

参数检查
   ↓
创建 Writer，进入队列
   ↓
成为 Leader，或等待其他线程协助
   ↓
PreprocessWrite
   ↓
组织 WriteGroup，计算序列号
   ↓
WriteGroupToWAL
   ↓
WriteBatchInternal::InsertInto
   ↓
SetLastSequence
   ↓
结束写入组，返回 Status

接下来若要追踪“key/value 到底如何放进 MemTable”，应该继续看 WriteBatchInternal::InsertInto()；若要理解多个线程如何选 leader、等待和唤醒，则继续看 WriteThread::JoinBatchGroup()。