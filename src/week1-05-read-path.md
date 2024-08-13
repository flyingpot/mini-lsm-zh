
# 读取路径

![章节概览](./lsm-tutorial/week1-05-overview.svg)

在本章中，您将：

* 将 SST 集成到 LSM 读取路径中。
* 使用 SST 实现 LSM 读取路径的 `get` 方法。
* 使用 SST 实现 LSM 读取路径的 `scan` 方法。

要复制测试用例到起始代码并运行它们，

```
cargo x copy-test --week 1 --day 5
cargo x scheck
```

## 任务 1：双合并迭代器

在此任务中，您需要修改：

```
src/iterators/two_merge_iterator.rs
```

您已经实现了一个合并迭代器，该迭代器合并相同类型的迭代器（即 memtable 迭代器）。现在我们已经实现了 SST 格式，我们既有磁盘上的 SST 结构，也有内存中的 memtable。当我们从存储引擎扫描时，我们需要将来自 memtable 迭代器和 SST 迭代器的数据合并到一个迭代器中。在这种情况下，我们需要一个 `TwoMergeIterator`，它可以合并两种不同类型的迭代器。

您可以在 `two_merge_iter.rs` 中实现 `TwoMergeIterator`。由于这里只有两个迭代器，我们不需要维护一个二叉堆。相反，我们可以简单地使用一个标志来指示从哪个迭代器读取。与 `MergeIterator` 类似，如果两个迭代器中都找到了相同的键，则第一个迭代器优先。

## 任务 2：读取路径 - 扫描

在此任务中，您需要修改：

```
src/lsm_iterator.rs
src/lsm_storage.rs
```

在实现 `TwoMergeIterator` 之后，我们可以将 `LsmIteratorInner` 的类型改为：

```rust,no_run
type LsmIteratorInner =
    TwoMergeIterator, MergeIterator>;
```

这样，LSM 存储引擎的内部迭代器将是一个结合了 memtable 和 SST 数据的迭代器。

请注意，我们的 SST 迭代器不支持传递结束边界。因此，您需要在 `LsmIterator` 中手动处理 `end_bound`。您需要修改 `LsmIterator` 逻辑，当内部迭代器的键达到结束边界时停止。

我们的测试用例将在 `l0_sstables` 中生成一些 memtable 和 SST，您需要在本任务中正确扫描所有这些数据。在下一章之前，您不需要刷新 SST。因此，您可以继续修改 `LsmStorageInner::scan` 接口，创建一个合并迭代器遍历所有 memtable 和 SST，以完成存储引擎的读取路径。

因为 `SsTableIterator::create` 涉及 I/O 操作，可能较慢，我们不希望在 `state` 关键部分执行此操作。因此，您应该首先读取 `state` 并克隆 LSM 状态快照的 `Arc`。然后，您应该释放锁。之后，您可以遍历所有 L0 SST，为每个 SST 创建迭代器，然后创建一个合并迭代器来检索数据。

```rust,no_run
fn scan(&self) {
    let snapshot = {
        let guard = self.state.read();
        Arc::clone(&guard)
    };
    // 创建迭代器并定位它们
}
```

在 LSM 存储状态中，我们仅在 `l0_sstables` 向量中存储 SST id。您需要从 `sstables` 哈希映射中检索实际的 SST 对象。

## 任务 3：读取路径 - Get

在此任务中，您需要修改：

```
src/lsm_storage.rs
```

对于 get 请求，它将作为在 memtable 中的查找，然后在 SST 上进行扫描。您可以在所有 SST 上创建一个合并迭代器，并在用户要查找的键上进行定位。定位有两种可能性：键与用户查找的相同，或者键不同/不存在。只有在键存在且与查找的键相同时，才应将值返回给用户。您还应减少状态锁的关键部分，如前一节所述。同时记得处理已删除的键。

## 测试您的理解

* 考虑用户有一个迭代器遍历整个存储引擎的情况，存储引擎大小为 1TB，因此扫描所有数据大约需要 1 小时。如果用户这样做会有什么问题？（这是一个很好的问题，我们将在教程的不同阶段多次询问它...）
* 一些 LSM 树存储引擎提供的另一个流行接口是 multi-get（或 vectored get）。用户可以传递他们想要检索的键列表。接口返回每个键的值。例如，`multi_get(vec!["a", "b", "c", "d"]) -> a=1,b=2,c=3,d=4`。显然，一个简单的实现是对每个键进行单个 get。您将如何实现 multi-get 接口，以及您可以进行哪些优化以使其更高效？（提示：在 get 过程中的一些操作只需要对所有键执行一次，除此之外，您可以考虑改进磁盘 I/O 接口以更好地支持此 multi-get 接口）。

我们没有提供这些问题的参考答案，欢迎在 Discord 社区中讨论。

## 额外任务

* **动态分派的代价。** 实现一个 `Box` 版本的合并迭代器，并进行基准测试以查看性能差异。
* **并行定位。** 创建合并迭代器需要加载所有底层 SST 的第一个块（当您创建 `SSTIterator` 时）。您可以并行化创建迭代器的过程。

{{#include copyright.md}}
