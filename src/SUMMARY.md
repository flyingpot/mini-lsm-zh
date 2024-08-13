
# 一周学完LSM

[前言](./00-preface.md)
[Mini-LSM概述](./00-overview.md)
[环境设置](./00-get-started.md)

- [第一周概述：Mini-LSM](./week1-overview.md)
  - [Memtable](./week1-01-memtable.md)
  - [合并迭代器](./week1-02-merge-iterator.md)
  - [块](./week1-03-block.md)
  - [排序字符串表（SST）](./week1-04-sst.md)
  - [读路径](./week1-05-read-path.md)
  - [写路径](./week1-06-write-path.md)
  - [小贴士：SST优化](./week1-07-sst-optimizations.md)

- [第二周概述：压缩与持久化](./week2-overview.md)
  - [压缩实现](./week2-01-compaction.md)
  - [简单压缩策略](./week2-02-simple.md)
  - [分层压缩策略](./week2-03-tiered.md)
  - [层级压缩策略](./week2-04-leveled.md)
  - [清单](./week2-05-manifest.md)
  - [预写日志（WAL）](./week2-06-wal.md)
  - [小贴士：批量写入与校验和](./week2-07-snacks.md)

- [第三周概述：MVCC](./week3-overview.md)
  - [时间戳编码 + 重构](./week3-01-ts-key-refactor.md)
  - [快照 - Memtables与时间戳](./week3-02-snapshot-read-part-1.md)
  - [快照 - 事务API](./week3-03-snapshot-read-part-2.md)
  - [水印与GC](./week3-04-watermark.md)
  - [事务与OCC](./week3-05-txn-occ.md)
  - [可序列化快照隔离](./week3-06-serializable.md)
  - [小贴士：压缩过滤器](./week3-07-compaction-filter.md)
- [余生（待定）](./week4-overview.md)

---

# 已废弃的Mini-LSM v1

- [概述](./00-v1.md)
  - [在小块中存储键值对](./01-block.md)
  - [并将其制成SST](./02-sst.md)
  - [现在是时候合并所有内容了](./03-memtable.md)
  - [引擎启动](./04-engine.md)
  - [让我们在后台做些什么](./05-compaction.md)
  - [系统崩溃时要小心](./06-recovery.md)
  - [一个好的布隆过滤器让生活更轻松](./07-bloom-filter.md)
  - [节省一些空间，希望如此](./08-key-compression.md)
  - [下一步是什么](./09-whats-next.md)
