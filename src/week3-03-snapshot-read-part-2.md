
# 快照读取 - 引擎读取路径和事务 API

在本章中，您将：

* 基于前一章完成读取路径，以支持快照读取。
* 实现事务 API 以支持快照读取。
* 实现引擎恢复过程，以正确恢复提交时间戳。

最终，您的引擎将能够为用户提供存储键空间的一致视图。

在重构过程中，您可能需要根据需要将某些函数的签名从 `&self` 更改为 `self: &Arc`。

要运行测试用例，请执行以下操作：

```
cargo x copy-test --week 3 --day 3
cargo x scheck
```

**注意：完成本章后，您还需要通过 2.5 和 2.6 的测试用例。**

## 任务 1：带读取时间戳的 LSM 迭代器

本章的目标是实现类似以下功能：

```rust,no_run
let snapshot1 = engine.new_txn();
// 向引擎写入一些内容
let snapshot2 = engine.new_txn();
// 向引擎写入一些内容
snapshot1.get(/* ... */); // 我们可以检索引擎之前状态的一致快照
```

为此，我们可以在创建事务时记录读取时间戳（即最新的提交时间戳）。当我们在事务上进行读取操作时，我们只会读取小于或等于读取时间戳的所有键版本。

在本任务中，您需要修改：

```
src/lsm_iterator.rs
```

为此，您需要在 `LsmIterator` 中记录一个读取时间戳。

```rust,no_run
impl LsmIterator {
    pub(crate) fn new(
        iter: LsmIteratorInner,
        end_bound: Bound,
        read_ts: u64,
    ) -> Result {
        // ...
    }
}
```

并且您需要更改 LSM 迭代器 `next` 逻辑以找到正确的键。

## 任务 2：多版本扫描和获取

在本任务中，您需要修改：

```
src/mvcc.rs
src/mvcc/txn.rs
src/lsm_storage.rs
```

现在我们在 LSM 迭代器中有 `read_ts`，我们可以在事务结构上实现 `scan` 和 `get`，以便在存储引擎中的给定点读取数据。

我们建议您在 `LsmStorageInner` 结构中创建类似 `scan_with_ts(/* 原始参数 */, read_ts: u64)` 和 `get_with_ts` 的辅助函数（如有必要）。存储引擎上的原始 get/scan 应实现为创建一个事务（快照）并在此事务上进行 get/scan。调用路径如下：

```
LsmStorageInner::scan -> new_txn 和 Transaction::scan -> LsmStorageInner::scan_with_ts
```

要在 `LsmStorageInner::scan` 中创建事务，我们需要向事务构造函数提供一个 `Arc`。因此，我们可以将 `scan` 的签名从简单的 `&self` 更改为 `self: &Arc`，以便我们可以创建一个事务，例如 `let txn = self.mvcc().new_txn(self.clone(), /* ... */)`。

您还需要更改 `scan` 函数以返回 `TxnIterator`。我们必须确保用户迭代引擎时快照是活动的，因此 `TxnIterator` 存储快照对象。在 `TxnIterator` 内部，我们现在可以存储一个 `FusedIterator`。我们将在实现 OCC 时将其更改为其他内容。

目前您不需要实现 `Transaction::put/delete`，所有修改仍将通过引擎进行。

## 任务 3：在 SST 中存储最大时间戳

在本任务中，您需要修改：

```
src/table.rs
src/table/builder.rs
```

在 SST 编码中，您应在块元数据之后存储最大时间戳，并在加载 SST 时恢复它。这将帮助系统在恢复系统时决定最新的提交时间戳。

## 任务 4：恢复提交时间戳

现在我们在 SST 中有最大时间戳信息，在 WAL 中有时间戳信息，我们可以获得引擎启动前提交的最大时间戳，并在创建 `mvcc` 对象时使用该时间戳作为最新的提交时间戳。

如果未启用 WAL，您只需通过在 SST 中找到最大时间戳来计算最新的提交时间戳。如果启用了 WAL，您应该进一步迭代所有恢复的 memtable 并找到最大时间戳。

在本任务中，您需要修改：

```
src/lsm_storage.rs
```

我们没有为此部分提供测试用例。完成此部分后，您应该通过之前章节（包括 2.5 和 2.6）的所有持久性测试。

## 测试您的理解

* 到目前为止，我们假设我们的 SST 文件使用单调递增的 ID 作为文件名。使用 `___.sst` 作为 SST 文件名是否可行？这可能会有哪些潜在问题？
* 考虑事务/快照的替代实现。在我们的实现中，我们在迭代器和事务上下文中使用 `read_ts`，以便用户始终可以访问基于时间戳的数据库的一个版本的一致视图。是否可以将当前 LSM 状态直接存储在事务上下文中以获取一致快照？（即，所有 SST ID、它们的级别信息以及所有 memtable + ts）这样做有什么优缺点？如果引擎没有 memtable 会怎样？如果引擎运行在像 S3 对象存储这样的分布式存储系统上会怎样？
* 考虑您正在实现 MVCC Mini-LSM 引擎的备份工具。仅仅复制所有 SST 文件而不备份 LSM 状态是否足够？为什么？

我们不提供这些问题的参考答案，欢迎在 Discord 社区中讨论。

{{#include copyright.md}}
