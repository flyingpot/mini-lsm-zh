
# 存储引擎与块缓存



这是 Mini-LSM 教程的旧版本，我们将不再维护它。我们现在有一个更好的版本，这一章节现在是 [Mini-LSM 第一周第5天：读路径](./week1-05-read-path.md) 和 [Mini-LSM 第一周第6天：写路径](./week1-06-write-path.md) 的一部分。




在这一部分，你需要修改：

* `src/lsm_iterator.rs`
* `src/lsm_storage.rs`
* `src/table.rs`
* 其他使用 `SsTable::read_block` 的部分

你可以使用 `cargo x copy-test day4` 将我们提供的测试用例复制到初始代码目录中。完成这一部分后，使用 `cargo x scheck` 检查风格并运行所有测试用例。如果你想编写自己的测试用例，在 `table.rs` 中编写一个新的模块 `#[cfg(test)] mod user_tests { /* 你的测试用例 */ }`。记得删除你修改的模块顶部的 `#![allow(...)]`，以便 cargo clippy 可以实际检查风格。

## 任务1 - 插入和删除

在实现插入和删除之前，让我们回顾一下 LSM 树的工作原理。LSM 树的结构包括：

* 内存表：一个活跃的可变内存表和多个不可变的内存表。
* 预写日志：每个内存表对应一个 WAL。
* SST：内存表可以以 SST 格式刷新到磁盘。SST 分层组织。

在这一部分，我们只需要锁定，将条目（或墓碑）写入活跃的内存表。你可以在 `lsm_storage.rs` 中修改。

## 任务2 - 获取

要从 LSM 中获取一个值，我们可以简单地从活跃内存表、不可变内存表（从最新到最早）和所有 SST 中探测。为了减少临界区，我们可以持有读锁，将所有指向内存表和 SST 的指针从 `LsmStorageInner` 结构中复制出来，并在临界区外创建迭代器。创建迭代器和探测时要注意顺序。

## 任务3 - 扫描

要创建一个扫描迭代器 `LsmIterator`，你需要使用 `TwoMergeIterator` 来合并内存表上的 `MergeIterator` 和 SST 上的 `MergeIterator`。你可以在 `lsm_iterator.rs` 中实现这个。可选地，你可以实现 `FusedIterator`，这样如果用户不小心在迭代器失效后调用 `next`，底层迭代器不会 panic。

`TwoMergeIterator` 生成的键值对序列可能包含空值，这意味着该值已被删除。`LsmIterator` 应该过滤这些空值。还需要正确处理起始和结束边界。

## 任务4 - 同步

在这一部分，我们将在 `lsm_storage.rs` 中实现内存表并刷新到 L0 SST。与任务1一样，写操作直接进入活跃的可变内存表。一旦调用 `sync`，我们将 SST 刷新到磁盘分两步进行：

* 首先，将当前可变内存表移动到不可变内存表列表中，这样未来的请求不会进入当前内存表。创建一个新的内存表。所有这些应该在一个临界区中发生并阻塞所有读取。
* 然后，我们可以在不持有任何锁的情况下将内存表刷新到磁盘作为 SST 文件。
* 最后，在一个临界区中，移除内存表并将 SST 放入 `l0_tables`。

一次只能有一个线程进行同步，因此你应该使用互斥锁来确保这一要求。

## 任务5 - 块缓存

既然我们已经实现了 LSM 结构，我们可以开始向磁盘写入内容了！之前在 `table.rs` 中，我们实现了一个 `FileObject` 结构，但没有向磁盘写入任何内容。在这个任务中，我们将改变实现，以便：

* `read` 将从磁盘读取，不使用任何缓存，使用 `std::os::unix::fs::FileExt` 中的 `read_exact_at`。
* 文件的大小应存储在结构体内部，`size` 函数直接返回它。
* `create` 应将文件写入磁盘。通常你应该在该文件上调用 `fsync`。但这会大大降低单元测试的速度。因此，我们在第6天恢复之前不进行 fsync。
* `open` 在第6天恢复之前保持未实现状态。

之后，我们可以在 `SsTable` 上实现一个新的 `read_block_cached` 函数，以便我们可以利用块缓存来服务读取请求。初始化 `LsmStorage` 结构时，你应该使用 `moka-rs` 创建一个 4GB 大小的块缓存。块按 SST id + 块 id 缓存。使用 `try_get_with` 从缓存中获取块/在缓存未命中时填充缓存。如果有多个请求读取同一个块且缓存未命中，`try_get_with` 只会向磁盘发出一个读取请求并将结果广播给所有请求。

记得更改 `SsTableIterator` 以使用块缓存。

## 额外任务

* 你可能已经注意到，每次我们进行获取、插入或删除操作时，都需要获取保护 LSM 结构的读锁；如果我们想要刷新，则需要获取写锁。这可能会导致很多问题。一些锁实现是公平的，这意味着只要有写者在等待锁，任何读者都不能获取锁。因此，写者将等待最慢的读者完成其操作，然后才能真正开始工作。一种可能的优化是实现 `WriteBatch`。我们不需要立即将用户的请求写入内存表 + WAL。我们可以允许用户进行一批写入。
* 将块对齐到 4K 并使用直接 I/O。

{{#include copyright.md}}
