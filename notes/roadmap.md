# 学习路线

起点：C++ 只会基础语法和容器，并发零基础。
路线原则：先补 C++ 基础，再学并发，最后学存储系统。每个阶段都从 LevelDB 里挑一段**短小、独立**的代码当教材，并配一个自己动手的练习。

当前进度：**阶段 0**

---

## 阶段 0：跑起来（1-2 天）
- [ ] 读 `notes/00-map.md`，对整体结构有个印象（看不懂细节没关系）
- [ ] 跑通单元测试和 `db_bench`，看懂输出里的吞吐和延迟
- [ ] 写一个 20 行的小程序：打开 DB，Put、Get、Delete，然后链接 `libleveldb.a` 编译运行
- **学到**：CMake 构建、静态库链接、怎么读 benchmark 输出

## 阶段 1：现代 C++ 基本功（1-2 周）
| 教材 | 行数 | 学什么 |
|---|---|---|
| `include/leveldb/slice.h` | 117 | 指针 + 长度的"视图"类型、const 引用、不拥有内存意味着什么 |
| `include/leveldb/status.h` | 122 | 用返回值代替异常处理错误、拷贝与移动构造 |
| `util/coding.cc` | 156 | varint 编码、位运算、大小端、把整数写成字节 |
| `util/mutexlock.h` | 39 | **RAII**：构造时加锁、析构时解锁（先只学 RAII 思想，锁留到阶段 3） |
- **练习**：自己实现一个 `MySlice` 和 varint 编解码，并用 gtest 写测试
- **AI Infra 联系**：零拷贝的视图类型在 tokenizer、tensor 切片里随处可见

## 阶段 2：内存管理（1-2 周）
| 教材 | 学什么 |
|---|---|
| `util/arena.cc`（66 行） | 内存池、对齐、为什么批量分配比 new/delete 快 |
| `util/cache.cc:151` LRUCache | 手写哈希表 + 双向链表实现 LRU、**引用计数**（`:52`, `:176`） |
- **练习**：实现一个固定块大小的分配器，以及一个单线程的 LRU cache
- **AI Infra 联系**：vLLM 的 PagedAttention 就是"KV cache 的块分配器 + 引用计数 + 淘汰策略"，和这里是同一类问题

## 阶段 3：并发入门（2-3 周，最重要的阶段）
| 教材 | 学什么 |
|---|---|
| `port/port_stdcxx.h:51`, `:69` | Mutex、CondVar 是对 std 的封装 |
| `util/env_posix.cc:814`, `:835` | 一个后台线程 + 任务队列 = 最简单的线程池 |
| `db/db_impl.cc:1206-1250` | 写入队列：多个线程排队，队首合并提交（group commit） |
- **练习**：从 `std::thread` 起步，依次写：数据竞争的反例 → 用 mutex 修复 → 生产者-消费者队列 → 迷你线程池。全程开 `-fsanitize=thread`（TSan）检查
- **AI Infra 联系**：group commit 和推理服务里的 continuous batching 是同一个思路：攒一批再处理，摊薄固定开销

## 阶段 4：原子操作与内存序（1-2 周）
| 教材 | 学什么 |
|---|---|
| `db/skiplist.h`（380 行） | 跳表结构、`std::atomic`、acquire/release（`:155`, `:161`）、一写多读不加锁 |
- **练习**：先实现单线程跳表，再改成一写多读，并通过 `db/skiplist_test.cc` 风格的并发测试
- **AI Infra 联系**：无锁队列、无锁计数器在 GPU 调度、RPC 框架里很常见

## 阶段 5：磁盘格式与 IO（2 周）
- `db/log_writer.cc` / `log_reader.cc`：WAL、校验和、崩溃恢复
- `table/block_builder.cc`、`table/table_builder.cc`：SSTable 格式、前缀压缩
- `util/bloom.cc`：布隆过滤器
- **练习**：写一个简易 WAL，做到写入后 kill 掉进程，重启能恢复数据
- **AI Infra 联系**：checkpoint 的保存与恢复、训练数据格式（如 TFRecord）都是同样的思路

## 阶段 6：系统全貌（2-3 周）
- `db/db_impl.cc` 的读写主线、`db/version_set.cc` 的版本管理和 Compaction
- **练习**：用 perf 给 `db_bench` 的 fillrandom 和 readrandom 做 profile，找到热点函数并解释原因
- **结业项目**：从零写一个迷你 LSM KV（MemTable + WAL + SST + 简单 Compaction），代码量控制在 1000 行左右

## 阶段 7：工程能力（贯穿全程）
- 每个练习都要配 gtest 测试
- 学会用 sanitizer（ASan、TSan、UBSan）、perf、benchmark
- 读 LevelDB 的提交历史，看真实的 bug 是怎么被发现和修复的。比如 HEAD 上的 `2946558` 修的就是一个整数溢出
