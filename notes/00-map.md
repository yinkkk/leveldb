# 00 · 全局地图

> 基于 commit `2946558`。现在看不懂细节没关系，后面每个阶段都会回到这里。

## LevelDB 是什么
LevelDB 是 Google 的嵌入式 KV 存储库，核心代码约 1.7 万行，底层结构是 **LSM-tree**。核心思想：把随机写变成顺序写。

```
写入 ──► WAL 日志(顺序追加) ──► MemTable(内存 SkipList)
                                    │ 写满 4MB
                                    ▼
                              Immutable MemTable
                                    │ 后台刷盘
                                    ▼
磁盘:  L0  [sst][sst][sst]      ← 文件之间 key 范围可以重叠
       L1  [sst|sst|sst]        ← 从 L1 起同层有序、互不重叠
       L2  [sst|sst|sst|sst...]   每层容量约是上一层的 10 倍
       ...  直到 L6 (共 7 层)
```

## 目录分工
| 目录 | 职责 | 关键文件 |
|---|---|---|
| `include/leveldb/` | 对外 API | `db.h`, `options.h` |
| `db/` | 引擎核心 | `db_impl.cc`, `version_set.cc`, `memtable.cc`, `skiplist.h` |
| `table/` | SSTable 磁盘格式 | `table_builder.cc`, `block_builder.cc` |
| `util/` | 基础设施 | `arena.cc`, `cache.cc`, `bloom.cc`, `coding.cc` |
| `port/` | 平台抽象 | `port_stdcxx.h` |

## 主线 1：Put
1. `DB::Put`（`db/db_impl.cc:1489`）包装成 `WriteBatch`，交给 `DBImpl::Write`（`:1206`）
2. 写线程排队，队首线程用 `BuildBatchGroup`（`:1226`）把后面的请求合并成一批
3. `MakeRoomForWrite`（`:1331`）：L0 文件达到 8 个时每次写入睡 1ms，达到 12 个时停写；MemTable 写满就转为 `imm_`（`:1396`）
4. 写 WAL（`:1236`），写盘期间释放锁
5. 插入 MemTable（`:1245` → `db/memtable.cc:76`）

## 主线 2：Get
`DBImpl::Get`（`db/db_impl.cc:1121`）按从新到旧的顺序查找：`mem_`（`:1148`）→ `imm_`（`:1150`）→ 逐层查 SST（`:1153`）

## 值得深挖的设计点
- **删除也是写入**：`db/dbformat.h:54` 定义墓碑类型，序列号 + 类型打包成 8 字节（`db/dbformat.cc:15`）
- **跳表一写多读不加锁**：`db/skiplist.h:155`, `:161`, `:363-364`
- **Arena 内存池**：`util/arena.cc:9`, `:21`
- **前缀压缩**：`table/block_builder.cc:7-20`，每 16 个 key 一个重启点（`include/leveldb/options.h:106`）
- **两种 Compaction 触发方式**：`db/version_set.cc:1258-1259`

## 待回答的问题
1. 为什么 L0 的文件之间 key 可以重叠，L1 及以下却不行？
2. 写 WAL 时释放了锁，这时另一个线程调用 `Write` 会怎样？（看 `db_impl.cc:1214`）
3. 把 `skiplist.h:364` 的 release 改成 relaxed，会出什么问题？（阶段 4 再答）
4. 为什么慢速阶段是每次写睡 1ms，而不是直接停住？
