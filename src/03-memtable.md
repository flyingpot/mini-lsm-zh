
# Mem Table 和 合并迭代器



这是 Mini-LSM 教程的旧版本，我们将不再维护它。我们现在有了这个教程的更好版本，这一章节现在是 [Mini-LSM 第一周第一天：内存表](./week1-01-memtable.md) 和 [Mini-LSM 第一周第二天：合并迭代器](./week1-02-merge-iterator.md) 的一部分。




在这一部分，你需要修改：

* `src/iterators/merge_iterator.rs`
* `src/iterators/two_merge_iterator.rs`
* `src/mem_table.rs`

你可以使用 `cargo x copy-test day3` 将我们提供的测试用例复制到起始代码目录中。完成这一部分后，使用 `cargo x scheck` 检查风格并运行所有测试用例。如果你想编写自己的测试用例，在 `table.rs` 中编写一个新的模块 `#[cfg(test)] mod user_tests { /* 你的测试用例 */ }`。记得移除你修改的模块顶部的 `#![allow(...)]`，以便 cargo clippy 可以实际检查风格。

这是 LSM 树基本构建块的最后一部分。实现合并迭代器后，我们可以轻松地从数据结构的不同部分（内存表 + SST）合并数据，并获取所有数据的迭代器。在第四部分，我们将把这些内容组合在一起，制作一个真正的存储引擎。

## 任务 1 - 内存表

在本教程中，我们使用 [crossbeam-skiplist](https://docs.rs/crossbeam-skiplist) 作为内存表的实现。跳表类似于链表，其中数据存储在列表节点中，不会在内存中移动。与使用单个指针指向下一个元素不同，跳表中的节点包含多个指针，允许用户“跳过某些元素”，从而实现 `O(log n)` 的搜索、插入和删除。

在存储引擎中，用户将创建数据结构的迭代器。通常，一旦用户修改了数据结构，迭代器就会失效（这是 C++ STL 和 Rust 容器的情况）。然而，跳表允许我们在访问和修改数据结构的同时进行操作，因此可能会在并发访问时提高性能。有些论文认为跳表不好，但数据在内存中保持原位的好特性可以使实现对我们来说更容易。

在 `mem_table.rs` 中，你需要基于 crossbeam-skiplist 实现一个内存表。请注意，内存表仅支持 `get`、`scan` 和 `put`，而不支持 `delete`。删除操作表示为 tombstone `key -> 空值`，实际数据将在压缩过程中删除（第 5 天）。请注意，所有 `get`、`scan`、`put` 函数只需要 `&self`，这意味着我们可以并发调用这些操作。

## 任务 2 - 内存表迭代器

你现在可以为 `内存表` 实现一个迭代器 `内存表迭代器`。`内存表.iter(start, end)` 将创建一个迭代器，返回范围内 `start, end` 内的所有元素。这里，start 是 `std::ops::Bound`，包含 3 种变体：`Unbounded`、`Included(key)`、`Excluded(key)`。`std::ops::Bound` 的表达能力消除了记忆 API 是否有闭区间或开区间的需要。

请注意，`crossbeam-skiplist` 的迭代器与 skiplist 本身的生存期相同，这意味着我们总是需要在使用迭代器时提供生存期。这很难使用。你可以使用 `ouroboros` crate 创建一个自引用结构，擦除生存期。你会发现 [ouroboros 示例][ouroboros-example] 很有帮助。

[ouroboros-example]: https://github.com/joshua-maros/ouroboros/blob/main/examples/src/ok_tests.rs

```rust
pub struct 内存表迭代器 {
    /// 持有对跳表的引用，以便迭代器有效。
    map: Arc
    /// 然后迭代器的生存期应与 `内存表迭代器` 结构本身相同
    iter: SkipList::Iter<'this>
}
```

你还需要将 Rust 风格的迭代器 API 转换为我们的存储迭代器。在 Rust 中，我们使用 `next() -> Data`。但在本教程中，`next` 没有返回值，数据应通过 `key()` 和 `value()` 获取。你需要想办法实现这一点。


剧透：内存表迭代器结构

```rust
#[self_referencing]
pub struct 内存表迭代器 {
    map: Arc>,
    #[borrows(map)]
    #[not_covariant]
    iter: SkipMapRangeIter<'this>,
    item: (Bytes, Bytes),
}
```

我们有 `map` 作为对 skipmap 的引用，`iter` 作为结构的自引用项，`item` 作为迭代器中的最后一项。你可能想过使用类似 `iter::Peekable` 的东西，但它需要在获取键和值时使用 `&mut self`。因此，一种方法是（1）在初始化 `内存表迭代器` 时从迭代器获取元素，存储在 `item` 中（2）在调用 `next` 时，我们从内部迭代器的 `next` 获取元素，并将内部迭代器移动到下一个位置。



在这种设计中，你可能已经注意到，只要我们拥有迭代器对象，内存表就不能从内存中释放。在本教程中，我们假设用户操作是短期的，因此这不会引起大问题。请参阅额外任务以了解可能的改进。

你也可以考虑使用 [AgateDB 的 skiplist](https://github.com/tikv/agatedb/tree/master/skiplist) 实现，它避免了创建自引用结构的问题。

## 任务 3 - 合并迭代器

现在你有很多内存表和 SSTs，你可能想将它们合并以获取键的最新出现。在 `merge_iterator.rs` 中，我们有 `合并迭代器`，它是一个合并所有相同类型迭代器的迭代器。`new` 函数中索引位置较低的迭代器具有更高的优先级，也就是说，如果我们有：

```
iter1: 1->a, 2->b, 3->c
iter2: 1->d
iter: 合并迭代器::create(vec![iter1, iter2])
```

最终迭代器将产生 `1->a, 2->b, 3->c`。iter1 中的数据将覆盖其他迭代器中的数据。

你可以使用 `BinaryHeap` 来实现这个合并迭代器。请注意，你永远不应该在二叉堆中放入任何无效的迭代器。一个常见的陷阱是错误处理。例如，

```rust
let Some(mut inner_iter) = self.iters.peek_mut() {
    inner_iter.next()?; // <- 会导致问题
}
```

如果 `next` 返回错误（例如，由于磁盘故障、网络故障、校验和错误等），它就不再有效。然而，当我们离开 if 条件并将错误返回给调用者时，`PeekMut` 的