
# 快照读取 - Memtables 和时间戳

在本章中，您将：

* 重构您的 memtable/WAL 以存储键的多个版本。
* 实现新的引擎写入路径，为每个键分配一个时间戳。
* 使您的压缩过程能够处理多版本键。
* 实现新的引擎读取路径，返回键的最新版本。

在重构过程中，您可能需要根据需要将某些函数的签名从 `&self` 更改为 `self: &Arc`。

要运行测试用例，请执行以下操作：

```
cargo x copy-test --week 3 --day 2
cargo x scheck
```

**注意：完成本章后，您还需要通过所有 <= 2.4 的测试。**

## 任务 1：MemTable、预写日志和读取路径

在本任务中，您需要修改以下文件：

```
src/wal.rs
src/mem_table.rs
src/lsm_storage.rs
```

我们已经将引擎中的大多数键更改为 `KeySlice`，其中包含一个字节键和一个时间戳。然而，我们系统中的某些部分仍然没有考虑时间戳。在第一个任务中，您需要修改您的 memtable 和 WAL 实现以考虑时间戳。

首先，您需要更改存储在 memtable 中的 `SkipMap` 类型。

```rust,no_run
pub struct MemTable {
    // map: Arc>,
    map: Arc>, // Bytes -> KeyBytes
    // ...
}
```

之后，您可以继续修复所有编译错误以完成此任务。

**MemTable::get**

我们保留了 get 接口，以便测试用例仍然可以探测 memtable 中特定版本的键。完成此任务后，此接口不应在您的读取路径中使用。由于我们在跳表中存储了 `KeyBytes`（即 `(Bytes, u64)`），而用户探测的是 `KeySlice`（即 `(&[u8], u64)`），我们需要找到一种方法将后者转换为前者的引用，以便从跳表中检索数据。

为此，您可以使用不安全代码将 `&[u8]` 强制转换为静态的，并使用 `Bytes::from_static` 从静态切片创建一个字节对象。这是合理的，因为 `Bytes` 不会尝试释放切片内存，因为它被认为是静态的。


提示：将 u8 切片转换为 Bytes

```rust,no_run
Bytes::from_static(unsafe { std::mem::transmute(key.key_ref()) })
```



以前这不是问题，因为我们有 `Bytes` 和 `&[u8]`，其中 `Bytes` 实现了 `Borrow<[u8]>`。

**MemTable::put**

签名应更改为 `fn put(&self, key: KeySlice, value: &[u8])`，并且您需要在实现中将键切片转换为 `KeyBytes`。

**MemTable::scan**

签名应更改为 `fn scan(&self, lower: Bound, upper: Bound) -> MemTableIterator`。您需要将 `KeySlice` 转换为 `KeyBytes`，并将其用作 `SkipMap::range` 参数。

**MemTable::flush**

在将 memtable 刷新到 SST 时，您现在应该使用键的时间戳，而不是默认时间戳。

**MemTableIterator**

它现在应该存储 `(KeyBytes, Bytes)`，返回的键类型应该是 `KeySlice`。

**Wal::recover** 和 **Wal::put**

预写日志现在应该接受键切片而不是用户键切片。在序列化和反序列化 WAL 记录时，您应该将时间戳放入 WAL 文件中，并对时间戳和之前所有的字段进行校验和。

WAL 格式如下：

```
| key_len (exclude ts len) (u16) | key | ts (u64) | value_len (u16) | value | checksum (u32) |
```

**LsmStorageInner::get**

以前，我们将 `get` 实现为首先探测 memtables，然后扫描 SSTs。现在我们更改了 memtable 以使用新的 key-ts API，我们需要重新实现 `get` 接口。最简单的方法是创建一个合并迭代器，覆盖我们所拥有的所有内容——memtables、不可变 memtables、L0 SSTs 和其他级别的 SSTs，这与您在 `scan` 中所做的相同，只是我们对 SSTs 进行了布隆过滤器过滤。

**LsmStorageInner::scan**

您需要结合新的 memtable API，并将扫描范围设置为 `(user_key_begin, TS_RANGE_BEGIN)` 和 `(user_key_end, TS_RANGE_END)`。请注意，在处理排除边界时，您需要将迭代器正确地定位到下一个键（而不是当前相同时间戳的键）。

## 任务 2：写入路径

在本任务中，您需要修改以下文件：

```
src/lsm_storage.rs
```

我们在 `LsmStorageInner` 中有一个 `mvcc` 字段，其中包含我们在本周用于多版本并发控制所需的所有数据结构。当您打开一个目录并初始化存储引擎时，您需要创建该结构。

在您的 `write_batch` 实现中，您需要为写批处理中的所有键获取一个提交时间戳。您可以在逻辑开始时使用 `self.mvcc().latest_commit_ts() + 1` 获取时间戳，并在逻辑结束时使用 `self.mvcc().update_commit_ts(ts)` 更新下一个提交时间戳。为了确保所有写批处理具有不同的时间戳，并且新键位于旧键之上，您需要在函数开始时持有写锁 `self.mvcc().write_lock.lock()`，以便一次只有一个线程可以写入存储引擎。

## 任务 3：MVCC 压缩

在本任务中，您需要修改以下文件：

```
src/compact.rs
```

在前几章中，我们只保留键的最新版本，并在将键压缩到底层时删除键。使用 MVCC，我们现在有时间戳与键相关联，我们不能使用相同的压缩逻辑。

在本章中，您可以简单地删除删除键的逻辑。您可以暂时忽略 `compact_to_bottom_level`，并在压缩过程中保留键的所有版本。

此外，您需要以一种方式实现压缩算法，即具有不同时间戳的相同键放入同一个 SST 文件中，即使它超过了 SST 大小限制。这确保了如果在某个级别的 SST 中找到了一个键，它不会出现在该级别的其他 SST 文件中，从而简化了系统中许多部分的实现。

## 任务 4：LSM 迭代器

在本任务中，您需要修改以下文件：

```
src/lsm_iterator.rs
```

在前一章中，我们实现了 LSM 迭代器，将具有不同时间戳的相同键视为不同键。现在，我们需要重构 LSM 迭代器，以便仅返回从子迭代器检索到的键的最新版本。

您需要在迭代器中记录 `prev_key`。如果我们已经向用户返回了键的最新版本，我们可以跳过所有旧版本并继续下一个键。

此时，您应该通过前几章中的所有测试，除了持久性测试（2.5 和 2.6）。

## 测试您的理解

* MVCC 引擎中的 `get` 与您在第 2 周构建的引擎有何不同？
* 在第 2 周，当 `get` 时，您在找到键的第一个 memtable/level 处停止。在 MVCC 版本中，您可以这样做吗？
* 如何将 `KeySlice` 转换为 `&KeyBytes`？这是安全/合理的操作吗？
* 为什么我们需要在写入路径中持有写锁？

我们不提供这些问题的参考答案，欢迎在 Discord 社区中讨论。

## 额外任务

* **Memtable Get 的早期停止**。我们可以实现 `get` 如下：如果在 memtable 中找到键的版本，我们可以停止搜索。这同样适用于 SSTs。

{{#include copyright.md}}
</step3