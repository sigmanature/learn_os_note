* ** 相关函数:[`__filemap_get_folio`](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c/__filemap_get_folio.md) **

好的，让我们逐个解答你的问题，首先解释为什么在 `__filemap_get_folio` 中当 `xa_is_value(folio)` 为真时 `folio` 会被设置为 `NULL`，然后我们将深入探讨 `filemap_get_entry` 函数。

**1. 为什么在 `__filemap_get_folio` 中当 `xa_is_value(folio)` 为真时 `folio = NULL`？**

你是对的，Linux 关于 XArray API 的文档指出 `xa_is_value` 用于检查 XArray 条目是否为“值条目”。然而，在 `__filemap_get_folio` 和 `filemap_get_entry` 的上下文中，“值条目”并不代表有效、可用的 folio。相反，它表示 XArray 中一种特殊的条目，它 *不是* 指向 `folio` 的指针。

以下是详细解释：

*   **XArray 值条目：** XArray 被设计为高效地存储指向对象的指针和小整数值，直接存储在数组结构本身中。这些小整数值被称为“值条目”。它们是一种性能优化，可以避免在简单情况下进行额外的内存分配。你可以使用诸如 `xa_mk_value()` 之类的函数创建这些值条目。

*   **`filemap_get_entry` 和值条目：** `filemap_get_entry` 函数（由 `__filemap_get_folio` 调用）旨在从页缓存的 XArray（或旧内核中的基数树）中检索条目。根据 `filemap_get_entry` 定义中的注释，它可以返回：
    *   **`folio` 指针：** 如果在给定索引处的页缓存中找到有效的 folio。
    *   **“影子条目” (shadow entry)：** 这是一种特殊条目，表示最近从页缓存中驱逐的 folio。它在正常意义上不是可用的 folio。
    *   **“交换条目” (swap entry)：** 在 `shmem/tmpfs` 的上下文中，这可能是一个表示已交换到交换空间的页面的条目。同样，也不是内存中直接可用的 folio。
    *   **`NULL`：** 如果在给定索引处未找到任何内容。

*   **`__filemap_get_folio` 中的 `xa_is_value(folio)`：** 当 `filemap_get_entry` 返回一个条目时，`__filemap_get_folio` 检查 `xa_is_value(folio)` 是否为真。如果为真，则意味着从 XArray 检索到的条目是**值条目**，并且在页缓存的上下文中，此值条目被解释为**不是有效、可用的 folio**。它很可能是一个影子条目或一些其他特殊标记。

*   **设置 `folio = NULL`：** 因此，如果 `xa_is_value(folio)` 为真，`__filemap_get_folio` 会设置 `folio = NULL`。这是因为该函数旨在返回一个 *可用* 的 `folio` 指针。如果它遇到值条目，则在页缓存上下文中它不是可用的 folio，因此将其视为在该索引处未找到 folio。然后，这会导致 `__filemap_get_folio` 中的 `if (!folio)` 条件，该条件将尝试创建新的 folio（如果设置了 `FGP_CREAT`）或返回错误 (`-ENOENT`)。

**本质上：** 在 `__filemap_get_folio` 中 `xa_is_value(folio)` 为真是一个信号，表明 `filemap_get_entry` 返回了一个特殊的非 folio 条目（如影子条目），并且 `__filemap_get_folio` 将其解释为在获取可用 folio 方面的缓存未命中。

**2. `filemap_get_entry(struct address_space *mapping, pgoff_t index)` 的详细解释**

现在，让我们逐行分解 `filemap_get_entry` 函数：

```c
void *filemap_get_entry(struct address_space *mapping, pgoff_t index)
{
	XA_STATE(xas, &mapping->i_pages, index);
	struct folio *folio;

	rcu_read_lock();
repeat:
	xas_reset(&xas);
	folio = xas_load(&xas);
	if (xas_retry(&xas, folio))
		goto repeat;
	/*
	 * A shadow entry of a recently evicted page, or a swap entry from
	 * shmem/tmpfs.  Return it without attempting to raise page count.
	 */
	if (!folio || xa_is_value(folio))
		goto out;

	if (!folio_try_get(folio))
		goto repeat;

	if (unlikely(folio != xas_reload(&xas))) {
		folio_put(folio);
		goto repeat;
	}
out:
	rcu_read_unlock();

	return folio;
}
```

*   **函数签名：**
    ```c
    void *filemap_get_entry(struct address_space *mapping, pgoff_t index)
    ```
    *   `void *`: 返回一个 void 指针。正如我们所见，这可以是 `struct folio *`、影子条目、交换条目或 `NULL`。
    *   `struct address_space *mapping`: 要在其中搜索的地址空间。
    *   `pgoff_t index`: 要查找的页缓存索引。

*   **XA_STATE 初始化：**
    ```c
    XA_STATE(xas, &mapping->i_pages, index);
    struct folio *folio;
    ```
    *   `XA_STATE(xas, &mapping->i_pages, index);`: 这会初始化一个名为 `xas` 的 `XA_STATE` 结构。 `XA_STATE` 用于管理 XArray 遍历的状态。
        *   `&mapping->i_pages`: `mapping->i_pages` 是 XArray（或基数树）的根，它存储此地址空间的页缓存条目。
        *   `index`: 我们正在搜索的索引。
    *   `struct folio *folio;`: 声明一个 `struct folio` 指针来存储检索到的 folio。

*   **RCU 读锁：**
    ```c
    rcu_read_lock();
    ```
    *   `rcu_read_lock();`: 获取 RCU (Read-Copy-Update) 读锁。 RCU 是一种同步机制，在许多情况下允许无锁读取。页缓存查找被设计为非常快速，并且在 RCU 下通常是无锁的。 RCU 读锁非常轻量级，并允许并发读取器。

*   **`repeat:` 标签和 XArray 查找循环：**
    ```c
repeat:
	xas_reset(&xas);
	folio = xas_load(&xas);
	if (xas_retry(&xas, folio))
		goto repeat;
    ```
    *   `repeat:`: 用于潜在重试的标签。
    *   `xas_reset(&xas);`: 重置 `XA_STATE` `xas` 以从根开始新的 XArray 查找。这对于重试很重要。
    *   `folio = xas_load(&xas);`: 这是核心的 XArray 查找操作。 `xas_load` 尝试从 `xas` 管理的 XArray 中加载给定 `index` 处的条目。它返回找到的条目（可以是 `folio` 指针、值条目或 `NULL`）。此查找被设计为非常快速，并且由于 RCU 通常是无锁的。
    *   `if (xas_retry(&xas, folio)) goto repeat;`: `xas_retry` 检查 `xas_load` 操作是否需要重试。这可能发生在并发场景中，其中 XArray 结构正在被修改。如果 `xas_retry` 返回 true，则意味着我们需要从头开始重试查找 (`goto repeat`)。

*   **影子/交换条目检查：**
    ```c
    /*
     * A shadow entry of a recently evicted page, or a swap entry from
     * shmem/tmpfs.  Return it without attempting to raise page count.
     */
    if (!folio || xa_is_value(folio))
        goto out;
    ```
    *   此部分检查我们是否获得了有效的 folio 或特殊条目。
    *   `if (!folio || xa_is_value(folio))`:
        *   `!folio`: 检查 `xas_load` 是否返回 `NULL`，这意味着在该索引处未找到任何内容。
        *   `xa_is_value(folio)`: 检查 `xas_load` 是否返回了值条目（如影子或交换条目）。
    *   `goto out;`: 如果这些条件中的任何一个为真，它将跳转到 `out` 标签，这将释放 RCU 读锁并返回 `folio`（如果 `!folio` 为真，则 `folio` 将为 `NULL`；如果 `xa_is_value(folio)` 为真，则 `folio` 将为值条目，但在 `__filemap_get_folio` 上下文中，值条目被视为 NULL）。**至关重要的是，在这些情况下，我们 *不* 尝试增加 folio 的引用计数，因为我们没有返回可用的 folio。**

*   **增加 Folio 引用计数：**
    ```c
    if (!folio_try_get(folio))
        goto repeat;
    ```
    *   `if (!folio_try_get(folio))`: 如果我们到达此处，则意味着 `folio` 不是 `NULL` 且不是值条目，因此它很可能是有效的 folio 指针。 `folio_try_get(folio)` 尝试原子地增加 folio 的引用计数。
    *   `goto repeat;`: 如果 `folio_try_get` 返回 false，则意味着它未能增加引用计数。如果 folio 正在被并发释放或从页缓存中删除，则可能发生这种情况。在这种情况下，我们需要从头开始重试整个查找过程 (`goto repeat`)，以获得新的、有效的 folio。

*   **重新加载和一致性检查：**
    ```c
    if (unlikely(folio != xas_reload(&xas))) {
        folio_put(folio);
        goto repeat;
    }
    ```
    *   `if (unlikely(folio != xas_reload(&xas)))`: `xas_reload(&xas)` *使用相同的 `XA_STATE`* 在 XArray 中执行另一次查找。这是一个至关重要的一致性检查。由于查找是在 RCU 读锁下完成的（非常轻量级），因此 XArray 结构可能在初始 `xas_load` *之后* 但在我们能够使用 `folio_try_get` 增加引用计数 *之前* 发生了更改。 `xas_reload` 确保我们获得的 folio 仍然是索引处的正确 folio。
    *   `folio != xas_reload(&xas)`: 比较我们最初获得的 folio (`folio`) 与我们在重新加载后获得的 folio (`xas_reload(&xas)`)。如果它们不同，则意味着 folio 可能在此期间已被替换或失效。
    *   `folio_put(folio);`: 如果 folio 不同，我们释放我们 *可能* 在 `folio_try_get` 中成功增加的引用计数（即使 `folio_try_get` 成功，folio 现在也可能无效）。
    *   `goto repeat;`: 从头开始重试整个查找过程 (`goto repeat`)，以获得页缓存的一致视图。

*   **`out:` 标签和 RCU 解锁：**
    ```c
out:
	rcu_read_unlock();
	return folio;
    ```
    *   `out:`: 当我们需要退出函数时到达的标签。
    *   `rcu_read_unlock();`: 释放 RCU 读锁。在获取 RCU 读锁后，始终释放它是至关重要的。
    *   `return folio;`: 返回 `folio`。这将是：
        *   有效的 `folio` 指针，如果我们成功找到 folio，增加了其引用计数，并通过了一致性检查。
        *   `NULL`，如果在缓存中未找到任何内容，或者如果我们遇到了影子/交换条目（由于 `if (!folio || xa_is_value(folio)) goto out;` 条件）。
        *   可能是值条目本身（尽管在 `__filemap_get_folio` 上下文中，值条目被视为 `NULL`）。

**总而言之，`filemap_get_entry` 函数执行以下操作：**

1.  **无锁查找：** 在 RCU 读锁下，在页缓存的 XArray 中执行非常快速、通常是无锁的查找。
2.  **处理特殊条目：** 识别并处理诸如影子条目和交换条目之类的特殊条目，而无需尝试将它们用作常规 folio。
3.  **引用计数：** 原子地增加找到的 folio 的引用计数，以确保在调用者使用它时它保持有效。
4.  **并发控制和一致性：** 使用 `folio_try_get` 和 `xas_reload` 进行重试，以处理对页缓存的并发修改，并确保返回的 folio 是一致且有效的。
5.  **返回 Folio 或 NULL：** 如果成功，则返回有效的 `folio` 指针；如果未找到可用的 folio，则返回 `NULL`。

此函数是一段高度优化且精心设计的代码，旨在为 Linux 页缓存提供高效且并发的访问。它是文件系统性能和内存管理的基石。

