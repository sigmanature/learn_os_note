**相关函数**
 *	[filemap_get_pages](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c/filemap_get_pages.md)
好的，让我们深入进行 `filemap_get_read_batch` 函数的非常详细的、逐步的解析。这个函数确实是当读取文件时，为了从页缓存中高效检索一批 folio 的核心组件。我们将逐行和逐概念地进行，以确保彻底的理解。

```c
static void filemap_get_read_batch(struct address_space *mapping,
		pgoff_t index, pgoff_t max, struct folio_batch *fbatch)
{
	XA_STATE(xas, &mapping->i_pages, index);
	struct folio *folio;

	rcu_read_lock();
	for (folio = xas_load(&xas); folio; folio = xas_next(&xas)) {
		if (xas_retry(&xas, folio))
			continue;
		if (xas.xa_index > max || xa_is_value(folio))
			break;
		if (xa_is_sibling(folio))
			break;
		if (!folio_try_get(folio))
			goto retry;

		if (unlikely(folio != xas_reload(&xas)))
			goto put_folio;

		if (!folio_batch_add(fbatch, folio))
			break;
		if (!folio_test_uptodate(folio))
			break;
		if (folio_test_readahead(folio))
			break;
		xas_advance(&xas, folio_next_index(folio) - 1);
		continue;
put_folio:
		folio_put(folio);
retry:
		xas_reset(&xas);
	}
	rcu_read_unlock();
}
```

**1. 函数签名和目的：**

```c
static void filemap_get_read_batch(struct address_space *mapping,
		pgoff_t index, pgoff_t max, struct folio_batch *fbatch)
```

*   **`static void filemap_get_read_batch(...)`**:
    *   `static`:  指示这个函数具有内部链接，意味着它仅在当前的 C 文件中可见。
    *   `void`:  函数不直接返回任何值。它修改作为参数传递的 `folio_batch` 结构体。
    *   `filemap_get_read_batch`: 名称清楚地表明了它的目的：为了从文件映射中读取而获取一批 folio。
*   **参数：**
    *   `struct address_space *mapping`:  指向 `address_space` 结构体的指针。这表示文件的地址空间，其中包含页缓存（由 xarray `i_pages` 表示）。
    *   `pgoff_t index`:  文件内我们想要开始获取 folio 的起始页索引（页偏移）。
    *   `pgoff_t max`:  我们应该获取 folio 的最大页索引（不包含），直到此索引为止。这定义了我们感兴趣的连续范围的上限。
    *   `struct folio_batch *fbatch`:  指向 `folio_batch` 结构体的指针。这个结构体用于累积检索到的 folio。函数会将找到的 folio 添加到这个批次中。

**目的总结：** `filemap_get_read_batch` 旨在从给定文件的页缓存中高效地检索连续的一批 folio，从 `index` 开始，直到（但不包括）`max`。它用这些 folio 填充提供的 `folio_batch` 结构体。

**2. `XA_STATE(xas, &mapping->i_pages, index);`**

```c
XA_STATE(xas, &mapping->i_pages, index);
```

*   **`XA_STATE(xas, &mapping->i_pages, index);`**: 这是一个宏调用，用于初始化一个名为 `xas` 的 `XA_STATE` 结构体。 `XA_STATE` 用于遍历 xarray（扩展数组），xarray 是 Linux 中用于高效管理页缓存（`mapping->i_pages`）的数据结构。
    *   `XA_STATE`:  一个宏，声明并初始化一个 `XA_STATE` 变量。可以将 `XA_STATE` 视为用于导航 xarray 的迭代器或游标。
    *   `xas`:  赋予 `XA_STATE` 变量的名称。使用 `xas` 作为 xarray 状态是一种常见的约定。
    *   `&mapping->i_pages`:  指向我们想要遍历的 xarray 的指针。在这种情况下，它是 `address_space` 结构体的 `i_pages` 成员，它保存了文件的页缓存。
    *   `index`:  xarray 内遍历的起始索引。我们想要从这个页索引开始查找 folio。

**本质上，这一行设置了 xarray 遍历状态 `xas`，以开始遍历 `mapping->i_pages` xarray，从页索引 `index` 开始。**

**3. `struct folio *folio;`**

```c
struct folio *folio;
```

*   **`struct folio *folio;`**:  声明一个名为 `folio` 的指针变量，类型为 `struct folio *`。这个变量将用于保存从遍历期间从 xarray 检索的每个 folio 的指针。

**4. `rcu_read_lock();` 和 `rcu_read_unlock();`**

```c
rcu_read_lock();
... // 循环和 xarray 操作
rcu_read_unlock();
```

*   **`rcu_read_lock();`**:  获取 RCU（Read-Copy-Update）读锁。
    *   **RCU (Read-Copy-Update):**  一种针对读取为主的场景优化的同步机制。它允许并发读取器和单个更新器，而无需为读取器进行昂贵的锁定。
    *   **`rcu_read_lock()`**:  进入 RCU 读取侧临界区。在这个区域内，读取 RCU 保护的数据结构（如此处的 xarray）是安全的，无需担心来自并发更新的数据损坏，只要您遵循 RCU 规则即可。至关重要的是，RCU 读锁非常轻量级，不会阻止写入器；写入器通过创建副本并以读取器看到一致视图的方式更新指针来继续进行。
*   **`rcu_read_unlock();`**:  释放 RCU 读锁，退出 RCU 读取侧临界区。

**此处 RCU 的目的：** 页缓存（xarray `mapping->i_pages`）是一个共享资源，可以被多个读取器（读取数据）和写入器（添加/删除页面，使缓存无效）并发访问。 RCU 读锁用于在 `filemap_get_read_batch` 中的读取操作期间保护 xarray。这允许多个 `filemap_get_read_batch` 调用并发运行，而不会相互阻塞或阻塞写入器，只要它们只是在读取即可。

**5. `for (folio = xas_load(&xas); folio; folio = xas_next(&xas))` - 主循环**

```c
for (folio = xas_load(&xas); folio; folio = xas_next(&xas)) {
    ... // 循环体
}
```

这是遍历 xarray 的核心循环。

*   **`folio = xas_load(&xas);`**:  这是 `for` 循环的初始化部分。
    *   `xas_load(&xas)`:  尝试从 xarray 中加载由 `xas` 状态指示的当前索引处的 folio。
    *   `folio = ...`:  `xas_load` 的结果（它是 `struct folio *` 或 `NULL`）被赋值给 `folio` 变量。
    *   **首次迭代：** 在首次迭代中，`xas_load` 将尝试加载起始索引 `index` 处的 folio（该索引用于初始化 `XA_STATE`）。如果该索引处存在 folio，则 `folio` 将指向它；否则，`folio` 将为 `NULL`。
*   **`folio;`**:  这是循环条件。只要 `folio` 不为 `NULL`，循环就继续。这意味着只要 `xas_load` 成功在当前 xarray 索引处找到 folio，循环就会迭代。
*   **`folio = xas_next(&xas)`**:  这是 `for` 循环的递增/下一步部分，在每次迭代结束时执行。
    *   `xas_next(&xas)`:  将 `xas` 状态前进到 xarray 中的 *下一个* 索引。它还尝试加载这个新索引处的 folio。
    *   `folio = ...`:  `xas_next` 的结果（`struct folio *` 或 `NULL`）被赋值回 `folio` 变量，为下一次迭代做准备。如果 `xas_next` 在下一个索引处找到 folio，则 `folio` 将指向它；否则，它将为 `NULL`，并且循环将在下一次条件检查中终止。

**总之，这个 `for` 循环旨在遍历 xarray，从初始 `index` 开始并顺序移动到更高的索引，在每次迭代中加载一个 folio。**

**6. `if (xas_retry(&xas, folio)) continue;`**

```c
if (xas_retry(&xas, folio))
    continue;
```

*   **`xas_retry(&xas, folio)`**:  检查 folio 加载操作是否需要重试。这与 RCU 和 xarray 的潜在并发修改有关。
    *   `xas_retry(...)`:  一个函数（可能是宏或内联函数），用于检查在 `xas_load` 或 `xas_next` 操作之后是否需要重试。
    *   `&xas`:  xarray 状态。
    *   `folio`:  刚刚加载的（或尝试加载的）folio。
    *   **RCU 和并发更新：** 在 RCU 保护的环境中，如果并发写入器在读取器遍历 xarray 时修改了 xarray，则读取器可能会暂时遇到不一致的状态。 `xas_retry` 检测到这种情况。
    *   **重试条件：** 如果 `xas_retry` 返回 true，则表示 folio 加载可能受到并发修改的影响，应该重试。
*   **`continue;`**:  如果 `xas_retry` 返回 true，则执行 `continue`，它会跳过当前循环迭代的其余部分，并转到下一次迭代（有效地在下一次迭代中重试 `xas_load` 或 `xas_next`）。

**`xas_retry` 的目的：** 处理由于 RCU 下的并发 xarray 修改而可能导致的不一致性。如果需要重试，则跳过当前 folio，并且 xarray 遍历有效地从下一次循环迭代中的相同索引重新开始，希望这次获得一致的视图。

**7. `if (xas.xa_index > max || xa_is_value(folio)) break;`**

```c
if (xas.xa_index > max || xa_is_value(folio))
    break;
```

这个 `if` 语句检查循环终止条件。

*   **`xas.xa_index > max`**:
    *   `xas.xa_index`:  `xas` 状态在 xarray 中正在检查的当前索引。
    *   `max`:  作为参数传递给 `filemap_get_read_batch` 的最大页索引（不包含）。
    *   **条件：** 如果当前 xarray 索引 `xas.xa_index` 超过 `max`，则表示我们已经超出了所需的 folio 范围。我们应该停止获取 folio 并跳出循环。
*   **`xa_is_value(folio)`**:
    *   `xa_is_value(folio)`:  检查加载的 `folio` 实际上是 xarray 中的特殊“值”条目，而不是指向真实 `folio` 结构体的指针。
    *   **Xarray 值：** Xarray 可以直接在 xarray 节点内存储特殊的“值”条目以进行优化（例如，对于小整数或标志）。这些不是真正的 folio。
    *   **条件：** 如果 `xa_is_value(folio)` 为 true，则表示我们在 xarray 中遇到了特殊的值条目，而不是 folio。我们应该停止获取 folio 并跳出循环，因为我们只对真正的 folio 感兴趣。
*   **`||` (OR 运算符):**  如果 *任何一个* 条件为真，则执行 `break;` 语句。
*   **`break;`**:  立即退出 `for` 循环。

**这个 `if` 条件的目的：** 当我们到达所需范围的末尾（`xas.xa_index > max`）或当我们遇到 xarray 中的非 folio 条目（`xa_is_value(folio)`）时，停止循环。

**8. `if (xa_is_sibling(folio)) break;`**

```c
if (xa_is_sibling(folio))
    break;
```

*   **`xa_is_sibling(folio)`**:  检查加载的 `folio` 是否为“同级” folio。
    *   **同级 Folio：** 在 xarray 和 folio 管理的上下文中，“同级” folio 可能指的是作为更大、复合 folio 结构一部分的 folio（例如，当使用更大的页面大小或 folio 聚合时）。确切含义可能取决于上下文，并且与特定的 folio 管理优化有关。
    *   **条件：** 如果 `xa_is_sibling(folio)` 为 true，则表明当前 folio 与其他 folio 以某种方式相关，我们不应在此批处理检索上下文中单独处理它。我们应该停止并跳出循环。
*   **`break;`**:  立即退出 `for` 循环。

**`xa_is_sibling` 检查的目的：** 处理 folio 以同级关系组织的情况（可能用于更大的页面支持或 folio 分组）。在 `filemap_get_read_batch` 中，我们可能对检索连续范围内单独的、非同级的 folio 感兴趣。

**9. `if (!folio_try_get(folio)) goto retry;`**

```c
if (!folio_try_get(folio))
    goto retry;
```

*   **`folio_try_get(folio)`**:  尝试增加 `folio` 的引用计数。
    *   **引用计数：** Folio，像许多内核对象一样，使用引用计数来管理其生命周期。 `folio_get(folio)`（或 `folio_try_get(folio)`）增加引用计数，表明现在有人正在使用该 folio，不应释放它。当使用完毕时，`folio_put(folio)` 会减少引用计数。
    *   **`folio_try_get(folio)`**:  非阻塞地尝试获取引用。它尝试增加引用计数。如果成功，则返回 true；如果失败（例如，如果 folio 正在被释放或处于不允许获取引用的状态），则返回 false。
*   **`!` (NOT 运算符):**  `!` 否定 `folio_try_get` 的结果。因此，如果 `folio_try_get` 返回 false（即，获取引用失败），则条件为真。
*   **`goto retry;`**:  如果 `folio_try_get` 失败，则跳转到 `retry` 标签。

**`folio_try_get` 和 `goto retry` 的目的：** 安全地获取 folio 的引用。如果 `folio_try_get` 失败，可能是由于并发的 folio 操作（例如，folio 正在失效或释放）。在这种情况下，我们无法可靠地使用此 folio。 `goto retry` 跳转到 `retry` 标签，这将重置 xarray 状态（`xas_reset(&xas)`），并有效地在下一次循环迭代中从当前索引重新开始 folio 检索过程，希望这次获得有效的 folio 引用。

**10. `if (unlikely(folio != xas_reload(&xas))) goto put_folio;`**

```c
if (unlikely(folio != xas_reload(&xas)))
    goto put_folio;
```

*   **`xas_reload(&xas)`**:  重新加载 xarray 中当前索引处的 folio，重要的是，它检查 folio 是否仍然是最初由 `xas_load` 或 `xas_next` 加载的 *同一个* folio。
    *   **并发检查：** `xas_reload` 对于 RCU 下的并发控制至关重要。在我们最初加载 `folio`（使用 `xas_load` 或 `xas_next`）到我们到达此检查之间的时间内，并发写入器 *可能* 已将 xarray 中此索引处的 folio 替换为 *不同的* folio（例如，由于页面替换或失效）。
    *   **返回值：** `xas_reload(&xas)` 返回 xarray 中索引处的 *当前* folio。
*   **`folio != xas_reload(&xas)`**:  比较我们最初获得的 `folio` 与 `xas_reload` 返回的 folio。如果它们 *不* 相同，则表示此索引处的 folio 已被并发替换。
*   **`unlikely(...)`**:  一个宏，向编译器提示内部条件预计在大多数情况下为假。这是一个优化提示。
*   **`goto put_folio;`**:  如果 folio 已被替换（`folio != xas_reload(&xas)`），则跳转到 `put_folio` 标签。

**`xas_reload` 和 `goto put_folio` 的目的：** 确保我们正在使用一致的 folio，并处理并发的 folio 替换。如果 `xas_reload` 检测到 folio 自我们最初加载以来已被替换，则表示我们的初始 folio 在当前上下文中不再有效。在这种情况下，我们必须释放我们可能已通过 `folio_try_get` 获取的引用（如果成功），并通过跳转到 `put_folio` 来丢弃此 folio。

**11. `if (!folio_batch_add(fbatch, folio)) break;`**

```c
if (!folio_batch_add(fbatch, folio))
    break;
```

*   **`folio_batch_add(fbatch, folio)`**:  尝试将当前 `folio` 添加到 `folio_batch` 结构体 `fbatch`。
    *   **`folio_batch` 结构体：** `folio_batch` 很可能是一个旨在高效保存 folio 集合的结构体。它可能具有固定大小的数组来存储 folio 指针，以及批次中当前 folio 的计数。
    *   **添加到批次：** `folio_batch_add` 尝试将 `folio` 添加到批次中。如果批次已满（已达到其最大容量），则可能会失败。
    *   **返回值：** 如果 folio 成功添加到批次中，`folio_batch_add` 很可能返回 true，如果失败（例如，批次已满），则返回 false。
*   **`!` (NOT 运算符):**  `!` 否定 `folio_batch_add` 的结果。因此，如果 `folio_batch_add` 返回 false（即，添加到批次失败），则条件为真。
*   **`break;`**:  如果 `folio_batch_add` 失败，则跳出循环。

**`folio_batch_add` 和 `break` 的目的：** 将检索到的 folio 添加到批次中。如果批次已满，我们无法添加更多 folio，因此我们停止获取并跳出循环。这限制了 folio 批次的大小。

**12. `if (!folio_test_uptodate(folio)) break;`**

```c
if (!folio_test_uptodate(folio))
    break;
```

*   **`folio_test_uptodate(folio)`**:  检查 `folio` 的 `uptodate` 标志是否已设置。
    *   **`uptodate` 标志：** 与 folio 关联的标志，指示 folio 的内容是否与底层存储（磁盘/SSD）一致。如果 `uptodate` 标志已设置，则表示 folio 包含有效、最新的数据。如果未设置，则 folio 可能包含过时或无效的数据，需要从存储中填充。
    *   **返回值：** 如果 `uptodate` 标志已设置，`folio_test_uptodate(folio)` 返回 true，否则返回 false。
*   **`!` (NOT 运算符):**  `!` 否定结果。因此，如果 `folio_test_uptodate(folio)` 返回 false（即，folio *不是* 最新的），则条件为真。
*   **`break;`**:  如果 folio 不是最新的，则跳出循环。

**`!folio_test_uptodate` 检查和 `break` 的目的：** 如果我们遇到不是最新的 folio，则停止获取 folio。在 `filemap_get_read_batch`（用于 *读取* 操作）的上下文中，我们可能对获取一批 *最新* 的 folio 感兴趣。如果我们找到不是最新的 folio，则可能表示缓存中连续的最新 folio 范围的结束，或者可能是停止批处理并单独处理非最新 folio 的信号（例如，通过启动 I/O 来填充它）。

**13. `if (folio_test_readahead(folio)) break;`**

```c
if (folio_test_readahead(folio))
    break;
```

*   **`folio_test_readahead(folio)`**:  检查 `folio` 的 `readahead` 标志是否已设置。
    *   **`readahead` 标志：** 可能在 folio 上设置的标志，指示它是作为预读操作的一部分带入缓存的。
    *   **返回值：** 如果 `readahead` 标志已设置，`folio_test_readahead(folio)` 返回 true，否则返回 false。
*   **`break;`**:  如果 `readahead` 标志已设置，则跳出循环。

**`folio_test_readahead` 检查和 `break` 的目的：** 这更加微妙。在 `folio_test_readahead` 上中断可能与批处理检索在整体读取路径中的使用方式有关。可能是：
    *   **“按需读取”范围的结束：** 设置了 `readahead` 标志的 Folio 可能是预取页面。 `filemap_get_read_batch` 可能旨在主要获取由于 *按需* 读取（直接请求）而进入缓存的 folio，并在到达预读区域时停止。
    *   **单独处理预读 Folio：** `filemap_get_read_batch` 的调用者可能有一种不同的方式来处理标记为 `readahead` 的 folio。通过在遇到预读 folio 时停止批处理，它允许调用者首先处理按需读取的 folio，然后可能单独处理预读 folio。
    *   **批处理策略：** 这可能是将仅“按需读取”的 folio 批处理在一起，并在不同的批处理或路径中处理预读 folio 的策略的一部分。

**14. `xas_advance(&xas, folio_next_index(folio) - 1);`**

```c
xas_advance(&xas, folio_next_index(folio) - 1);
```

*   **`xas_advance(&xas, ...)`**:  将 `xas` 状态前进到 xarray 中的新索引。这是一种在 xarray 遍历中向前跳跃的方法，而不是仅仅移动到紧邻的下一个索引（`xas_next` 执行此操作）。
*   **`folio_next_index(folio)`**:  返回逻辑上在连续序列中当前 `folio` 之后出现的 folio 的索引。例如，如果 `folio` 表示索引为 10、11、12、13 的页面（如果它是 4 页的 folio），则 `folio_next_index(folio)` 将返回 14。
*   **`folio_next_index(folio) - 1`**:  从 `folio_next_index(folio)` 中减去 1。乍一看这可能有点奇怪。但是，请考虑 `xas_next(&xas)` 的工作方式：在 `xas_advance` 之后，*下一次* 调用 `xas_next(&xas)` 将有效地从 `folio_next_index(folio)` 开始。通过前进到 `folio_next_index(folio) - 1`，然后依赖循环中的 `xas_next`，加载的 *下一个* folio 将是连续范围内 *紧随* 当前 `folio` 之后的 folio。

**`xas_advance` 的目的：** 有效地跳过当前 `folio` 覆盖的页面范围，并直接移动到连续序列中下一个潜在 folio 的索引。这是一种优化，可以避免遍历已被当前 folio 覆盖的索引，特别是如果 folio 可以表示多个页面。

**15. `continue;`**

```c
continue;
```

*   **`continue;`**:  在 `xas_advance` 之后，执行 `continue`。这会跳过当前循环迭代的其余部分，并转到 *下一个* 迭代，从 `folio = xas_load(&xas);` 部分开始。实际上，它移动到处理高级索引处的 folio。

**16. `put_folio:` 标签和 `folio_put(folio);`**

```c
put_folio:
    folio_put(folio);
```

*   **`put_folio:`**:  一个标签，通过 `if (unlikely(folio != xas_reload(&xas)))` 条件中的 `goto put_folio;` 跳转到该标签。
*   **`folio_put(folio);`**:  减少 `folio` 的引用计数。当我们决定不使用 folio 时（例如，因为它被并发替换），这对于释放 `folio_try_get(folio)` 获取的引用至关重要。

**`put_folio` 和 `folio_put` 的目的：** 当我们遇到无法使用当前加载的 folio 的情况时（例如，检测到并发替换），释放 folio 引用。必须平衡每个 `folio_get`（或 `folio_try_get`）与相应的 `folio_put`，以避免 folio 泄漏。

**17. `retry:` 标签和 `xas_reset(&xas);`**

```c
retry:
    xas_reset(&xas);
```

*   **`retry:`**:  一个标签，通过 `if (!folio_try_get(folio))` 条件中的 `goto retry;` 跳转到该标签。
*   **`xas_reset(&xas)`**:  重置 `xas` 状态。这可能会将 `xas` 状态的内部索引重置回其初始化的索引（或重置为某些先前的一致状态）。

**`retry` 和 `xas_reset` 的目的：** 处理 `folio_try_get` 失败的情况。在这种情况下，我们无法可靠地使用 folio。 `xas_reset(&xas)` 用于重置 xarray 遍历状态，然后循环将继续到下一次迭代。实际上，它会重试从下一次迭代中的相同索引加载 folio，希望导致 `folio_try_get` 失败的瞬时问题将得到解决。

**整体函数逻辑和批处理检索总结：**

`filemap_get_read_batch` 的工作方式如下：

1.  **初始化：** 设置 xarray 遍历状态 `xas`，从 `index` 开始。
2.  **RCU 保护：** 进入 RCU 读取侧临界区，以实现安全的 xarray 访问。
3.  **循环和 Folio 加载：** 使用 `xas_load` 和 `xas_next` 遍历 xarray，一次加载一个 folio。
4.  **重试机制：** 使用 `xas_retry` 处理潜在的 RCU 不一致性，并在需要时重试 folio 加载。
5.  **范围和类型检查：** 检查当前索引是否在所需范围内（`xas.xa_index > max`）、是否为值条目（`xa_is_value`）或同级 folio（`xa_is_sibling`），如果满足任何这些条件，则中断循环。
6.  **引用获取：** 尝试使用 `folio_try_get` 获取 folio 的引用。如果失败，则重试。
7.  **并发验证：** 使用 `xas_reload` 验证 folio 自最初加载以来是否已被并发替换。如果被替换，则丢弃 folio 并释放引用。
8.  **批处理添加：** 使用 `folio_batch_add` 将有效的 folio 添加到 `folio_batch`。如果批次已满，则停止。
9.  **最新和预读检查：** 检查 folio 是否为最新，以及是否设置了预读标志。如果满足任一条件（可能在最新或非预读 folio 的边界处停止），则中断循环。
10. **高效前进：** 使用 `xas_advance` 跳过当前 folio 覆盖的页面，并移动到连续范围内下一个潜在 folio。
11. **清理：** 在需要时使用 `folio_put` 释放 folio 引用，并使用 `xas_reset` 重置 xarray 状态以进行重试。
12. **RCU 解锁：** 退出 RCU 读取侧临界区。

**本质上，`filemap_get_read_batch` 是一个健壮且高效的函数，用于从页缓存中检索一批连续的、最新的和非预读的 folio，同时处理并发、错误和批处理大小限制，并优化 xarray 遍历。** 它是 Linux 内核中文件读取操作的关键构建块。


